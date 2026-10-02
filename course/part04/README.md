# Part 04: Functions พื้นฐาน
## ขั้นตอนที่ 51-70

---

## ขั้นตอนที่ 51: การประกาศและเรียกใช้ฟังก์ชัน

```kotlin
// ============================================
// รูปแบบการประกาศฟังก์ชัน
// ============================================

// แบบปกติ
fun greet(name: String): String {
    return "สวัสดี, $name!"
}

// Single expression function (ไม่ต้องมี { return ... })
fun greetShort(name: String): String = "สวัสดี, $name!"

// ไม่มี return type (Unit = void)
fun printMessage(msg: String) {
    println(msg)
}

// ระบุ Unit อย่างชัดเจน (ไม่บังคับแต่ใส่ได้)
fun printMessageExplicit(msg: String): Unit {
    println(msg)
}

// Multiple parameters
fun add(a: Int, b: Int): Int = a + b

// ============================================
// Named Arguments - ระบุชื่อ parameter
// ============================================

fun createProfile(
    name: String,
    age: Int,
    email: String,
    city: String = "กรุงเทพฯ"  // Default parameter
): String {
    return "ชื่อ: $name | อายุ: $age | Email: $email | เมือง: $city"
}

fun main() {
    // เรียกตามลำดับ
    println(greet("สมชาย"))
    println(greetShort("สมหญิง"))
    println(add(3, 4))
    
    // Named arguments - ไม่ต้องเรียงตามลำดับ
    println(createProfile(
        name = "สมชาย",
        age = 25,
        email = "somchai@example.com"
        // city ใช้ default value
    ))
    
    println(createProfile(
        email = "test@example.com",
        city = "เชียงใหม่",
        age = 30,
        name = "สมศักดิ์"
    ))
}
```

---

## ขั้นตอนที่ 52: Default Parameters

```kotlin
// ============================================
// Default Parameters
// ============================================

fun sendNotification(
    title: String,
    message: String,
    priority: Int = 0,
    vibrate: Boolean = false,
    sound: String = "default",
    badge: Int = 0
) {
    println("📬 Notification:")
    println("  Title: $title")
    println("  Message: $message")
    println("  Priority: $priority")
    println("  Vibrate: $vibrate")
    println("  Sound: $sound")
    println("  Badge: $badge")
}

// ============================================
// Overloading vs Default Parameters
// ============================================

// วิธีเก่า (Java style) - overloading
fun connect(host: String, port: Int, timeout: Int) { /* ... */ }
fun connect(host: String, port: Int) = connect(host, port, 30)
fun connect(host: String) = connect(host, 8080, 30)

// วิธีใหม่ (Kotlin style) - default parameters (ดีกว่า)
fun connectKotlin(
    host: String,
    port: Int = 8080,
    timeout: Int = 30,
    ssl: Boolean = false,
    retries: Int = 3
) {
    println("กำลังเชื่อมต่อ $host:$port (timeout=${timeout}s, ssl=$ssl, retries=$retries)")
}

fun main() {
    // ใช้ default ทั้งหมด
    sendNotification("แจ้งเตือน", "คุณมีข้อความใหม่")
    
    println()
    
    // ระบุบางค่า
    sendNotification(
        title = "ด่วน!",
        message = "กรุณาตรวจสอบระบบ",
        priority = 2,
        vibrate = true
    )
    
    println()
    
    // Connect examples
    connectKotlin("api.example.com")
    connectKotlin("db.example.com", port = 5432)
    connectKotlin("secure.example.com", port = 443, ssl = true)
    connectKotlin("api.example.com", 8080, 60, false, 5)
}
```

---

## ขั้นตอนที่ 53: Vararg (Variable Arguments)

```kotlin
// ============================================
// vararg - รับ argument ได้ไม่จำกัด
// ============================================

fun sum(vararg numbers: Int): Int {
    var total = 0
    for (n in numbers) total += n
    return total
}

fun max(vararg numbers: Int): Int {
    if (numbers.isEmpty()) throw IllegalArgumentException("ต้องมีอย่างน้อย 1 ตัวเลข")
    return numbers.max()
}

fun printAll(vararg items: Any?) {
    items.forEach { println(it) }
}

fun createTable(header: String, vararg rows: String): String {
    val separator = "-".repeat(40)
    return buildString {
        appendLine(separator)
        appendLine("| $header".padEnd(39) + "|")
        appendLine(separator)
        rows.forEach { row ->
            appendLine("| $row".padEnd(39) + "|")
        }
        appendLine(separator)
    }
}

// ============================================
// Spread Operator (*) - ส่ง Array เป็น vararg
// ============================================

fun main() {
    println("sum(1,2,3) = ${sum(1, 2, 3)}")
    println("sum(1,2,3,4,5) = ${sum(1, 2, 3, 4, 5)}")
    println("sum() = ${sum()}")
    
    println("max(3,1,4,1,5,9,2,6) = ${max(3, 1, 4, 1, 5, 9, 2, 6)}")
    
    println()
    printAll("Hello", 42, true, null, 3.14)
    
    println()
    println(createTable(
        "รายการสินค้า",
        "1. มะม่วง - 50 บาท",
        "2. กล้วย - 30 บาท",
        "3. ส้ม - 40 บาท"
    ))
    
    // Spread operator
    val numbers = intArrayOf(1, 2, 3, 4, 5)
    println("sum of array = ${sum(*numbers)}")  // spread array
    
    // ผสม spread กับ literal
    val moreNumbers = intArrayOf(6, 7, 8)
    println("combined = ${sum(0, *numbers, *moreNumbers, 9)}")
}
```

---

## ขั้นตอนที่ 54: Return Types และ Nothing

```kotlin
// ============================================
// Return Types ต่างๆ
// ============================================

// Return Unit (void)
fun logMessage(msg: String): Unit {
    println("[LOG] $msg")
}

// Return ค่า
fun double(n: Int): Int = n * 2

// Return nullable
fun findUser(id: Int): String? {
    val users = mapOf(1 to "สมชาย", 2 to "สมหญิง")
    return users[id]  // อาจ return null
}

// ============================================
// Nothing - ฟังก์ชันที่ไม่มีวันสำเร็จ
// ============================================

// Nothing ใช้เมื่อ:
// 1. ฟังก์ชัน throw exception เสมอ
fun fail(message: String): Nothing {
    throw IllegalStateException(message)
}

// 2. Infinite loop
fun infiniteLoop(): Nothing {
    while (true) { /* loop forever */ }
}

// ============================================
// Practical use of Nothing
// ============================================

fun getConfigValue(key: String): String {
    val config = mapOf(
        "API_URL" to "https://api.example.com",
        "API_KEY" to "secret123"
    )
    return config[key] ?: fail("Config key '$key' ไม่พบ")
    // Kotlin รู้ว่า fail() ไม่ return ดังนั้น ?: จึงไม่จำเป็นต้อง match type
}

fun requirePositive(n: Int): Int {
    return if (n > 0) n
    else fail("ค่าต้องมากกว่า 0 แต่ได้ $n")
}

fun main() {
    logMessage("โปรแกรมเริ่มต้น")
    
    println(double(5))
    
    println(findUser(1))   // สมชาย
    println(findUser(99))  // null
    
    // Safe access
    val user = findUser(1) ?: "ไม่พบผู้ใช้"
    println("User: $user")
    
    try {
        println(getConfigValue("API_URL"))
        println(getConfigValue("MISSING_KEY"))  // จะ throw
    } catch (e: IllegalStateException) {
        println("Error: ${e.message}")
    }
    
    try {
        println(requirePositive(5))
        println(requirePositive(-3))  // จะ throw
    } catch (e: IllegalStateException) {
        println("Error: ${e.message}")
    }
}
```

---

## ขั้นตอนที่ 55: Local Functions

```kotlin
// ============================================
// Local Functions - ฟังก์ชันภายในฟังก์ชัน
// ============================================

fun processData(data: List<String>): List<String> {
    // Local function - เข้าถึง outer scope ได้
    fun isValid(item: String): Boolean {
        return item.isNotBlank() && item.length >= 3
    }
    
    fun normalize(item: String): String {
        return item.trim().lowercase()
    }
    
    fun capitalize(item: String): String {
        return item.replaceFirstChar { it.uppercase() }
    }
    
    return data
        .filter { isValid(it) }
        .map { normalize(it) }
        .map { capitalize(it) }
        .distinct()
        .sorted()
}

// ตัวอย่าง: Validation ด้วย local functions
fun validateRegistration(
    username: String,
    email: String,
    password: String
): List<String> {
    val errors = mutableListOf<String>()
    
    fun validateUsername() {
        when {
            username.isBlank() -> errors.add("ชื่อผู้ใช้ไม่สามารถว่างได้")
            username.length < 3 -> errors.add("ชื่อผู้ใช้ต้องมีอย่างน้อย 3 ตัวอักษร")
            username.length > 20 -> errors.add("ชื่อผู้ใช้ต้องไม่เกิน 20 ตัวอักษร")
            !username.matches(Regex("[a-zA-Z0-9_]+")) ->
                errors.add("ชื่อผู้ใช้ใช้ได้แค่ตัวอักษร ตัวเลข และ _")
        }
    }
    
    fun validateEmail() {
        val emailRegex = Regex("^[A-Za-z0-9+_.-]+@[A-Za-z0-9.-]+\\.[A-Za-z]{2,}$")
        when {
            email.isBlank() -> errors.add("Email ไม่สามารถว่างได้")
            !email.matches(emailRegex) -> errors.add("รูปแบบ Email ไม่ถูกต้อง")
        }
    }
    
    fun validatePassword() {
        when {
            password.length < 8 -> errors.add("รหัสผ่านต้องมีอย่างน้อย 8 ตัวอักษร")
            !password.any { it.isDigit() } -> errors.add("รหัสผ่านต้องมีตัวเลขอย่างน้อย 1 ตัว")
            !password.any { it.isUpperCase() } -> errors.add("รหัสผ่านต้องมีตัวพิมพ์ใหญ่อย่างน้อย 1 ตัว")
            !password.any { it.isLowerCase() } -> errors.add("รหัสผ่านต้องมีตัวพิมพ์เล็กอย่างน้อย 1 ตัว")
        }
    }
    
    validateUsername()
    validateEmail()
    validatePassword()
    
    return errors
}

fun main() {
    val rawData = listOf("  HELLO  ", "world", "hi", "  kotlin  ", "WORLD", "", "  ", "abc")
    println("Processed: ${processData(rawData)}")
    
    println("\nValidation Tests:")
    
    val testCases = listOf(
        Triple("ab", "invalid-email", "weak"),
        Triple("validuser", "user@test.com", "StrongPass1"),
        Triple("john_doe_123", "john@example.com", "SecureP@ss1")
    )
    
    testCases.forEach { (username, email, password) ->
        println("\nTest: username='$username', email='$email'")
        val errors = validateRegistration(username, email, password)
        if (errors.isEmpty()) {
            println("  ✅ สมัครสมาชิกสำเร็จ!")
        } else {
            println("  ❌ พบข้อผิดพลาด:")
            errors.forEach { println("     - $it") }
        }
    }
}
```

---

## ขั้นตอนที่ 56: Infix Functions

```kotlin
// ============================================
// Infix Functions - เรียกใช้แบบ infix notation
// ============================================

// infix function ต้องเป็น member function หรือ extension function
// และต้องมี parameter เดียวเท่านั้น

infix fun Int.plus(other: Int): Int = this + other

data class Point(val x: Int, val y: Int) {
    infix fun distanceTo(other: Point): Double {
        val dx = (this.x - other.x).toDouble()
        val dy = (this.y - other.y).toDouble()
        return Math.sqrt(dx * dx + dy * dy)
    }
    
    infix fun moveBy(delta: Point): Point =
        Point(this.x + delta.x, this.y + delta.y)
}

// ใช้ infix สำหรับ DSL-like syntax
data class Range(val start: Int, val end: Int)

infix fun Int.to(end: Int): Range = Range(this, end)

infix fun Range.step(stepSize: Int): List<Int> {
    val result = mutableListOf<Int>()
    var current = start
    while (current <= end) {
        result.add(current)
        current += stepSize
    }
    return result
}

// Infix สำหรับ test assertions
infix fun <T> T.shouldBe(expected: T) {
    if (this != expected) {
        throw AssertionError("Expected $expected but got $this")
    }
    println("✅ $this == $expected")
}

fun main() {
    // Infix call
    val sum = 3 plus 4  // เหมือน 3.plus(4)
    println("3 plus 4 = $sum")
    
    val p1 = Point(0, 0)
    val p2 = Point(3, 4)
    
    // Infix call
    val distance = p1 distanceTo p2  // เหมือน p1.distanceTo(p2)
    println("ระยะทาง $p1 ถึง $p2 = $distance")
    
    val p3 = p1 moveBy Point(2, 3)
    println("$p1 ย้ายไป Point(2,3) = $p3")
    
    // DSL-like
    val myRange = 1 to 10 step 2
    println("1 to 10 step 2 = $myRange")
    
    // Assertions
    println("\nAssertions:")
    5 + 3 shouldBe 8
    "Hello".length shouldBe 5
    listOf(1, 2, 3).size shouldBe 3
}
```

---

## ขั้นตอนที่ 57: Tail Recursive Functions

```kotlin
// ============================================
// tailrec - Tail Recursive Optimization
// ============================================

// ปกติ - อาจ StackOverflow กับ n ขนาดใหญ่
fun factorialNormal(n: Long): Long {
    return if (n <= 1) 1L
    else n * factorialNormal(n - 1)
}

// tailrec - Compiler optimize เป็น loop อัตโนมัติ
tailrec fun factorialTailrec(n: Long, accumulator: Long = 1L): Long {
    return if (n <= 1) accumulator
    else factorialTailrec(n - 1, n * accumulator)
}

// Fibonacci แบบ tailrec
tailrec fun fibonacci(n: Int, a: Long = 0, b: Long = 1): Long {
    return when (n) {
        0 -> a
        1 -> b
        else -> fibonacci(n - 1, b, a + b)
    }
}

// Sum ของ list ด้วย tailrec
tailrec fun sumList(list: List<Int>, accumulator: Int = 0): Int {
    if (list.isEmpty()) return accumulator
    return sumList(list.drop(1), accumulator + list.first())
}

// GCD (Greatest Common Divisor) ด้วย tailrec
tailrec fun gcd(a: Int, b: Int): Int =
    if (b == 0) a else gcd(b, a % b)

fun main() {
    // Factorial
    println("Factorial:")
    for (n in 0L..10L) {
        println("  $n! = ${factorialTailrec(n)}")
    }
    
    // ทดสอบกับ n ขนาดใหญ่ (tailrec ไม่ stack overflow)
    println("  20! = ${factorialTailrec(20)}")
    
    // Fibonacci
    println("\nFibonacci:")
    for (n in 0..15) {
        print("${fibonacci(n)} ")
    }
    println()
    
    // GCD
    println("\nGCD:")
    println("gcd(48, 18) = ${gcd(48, 18)}")  // 6
    println("gcd(100, 75) = ${gcd(100, 75)}") // 25
    println("gcd(17, 13) = ${gcd(17, 13)}")   // 1
    
    // Sum list
    val bigList = (1..100).toList()
    println("\nSum 1-100 = ${sumList(bigList)}")  // 5050
}
```

---

## ขั้นตอนที่ 58: Operator Overloading

```kotlin
// ============================================
// Operator Overloading
// ============================================

data class Vector2D(val x: Double, val y: Double) {
    // + operator
    operator fun plus(other: Vector2D): Vector2D =
        Vector2D(x + other.x, y + other.y)
    
    // - operator
    operator fun minus(other: Vector2D): Vector2D =
        Vector2D(x - other.x, y - other.y)
    
    // * operator (scalar multiply)
    operator fun times(scalar: Double): Vector2D =
        Vector2D(x * scalar, y * scalar)
    
    // / operator (scalar divide)
    operator fun div(scalar: Double): Vector2D =
        Vector2D(x / scalar, y / scalar)
    
    // unary - operator
    operator fun unaryMinus(): Vector2D = Vector2D(-x, -y)
    
    // == operator (data class จัดการให้อัตโนมัติ)
    
    val magnitude: Double get() = Math.sqrt(x * x + y * y)
    
    fun normalized(): Vector2D = this / magnitude
    
    fun dot(other: Vector2D): Double = x * other.x + y * other.y
    
    override fun toString(): String = "Vector2D(%.2f, %.2f)".format(x, y)
}

// Operator overloading สำหรับ Matrix
data class Matrix2x2(
    val a: Double, val b: Double,
    val c: Double, val d: Double
) {
    operator fun plus(other: Matrix2x2) = Matrix2x2(
        a + other.a, b + other.b,
        c + other.c, d + other.d
    )
    
    operator fun times(other: Matrix2x2) = Matrix2x2(
        a * other.a + b * other.c,
        a * other.b + b * other.d,
        c * other.a + d * other.c,
        c * other.b + d * other.d
    )
    
    val determinant: Double get() = a * d - b * c
    
    override fun toString(): String = 
        "[${"%.1f".format(a)} ${"%.1f".format(b)}]\n[${"%.1f".format(c)} ${"%.1f".format(d)}]"
}

fun main() {
    val v1 = Vector2D(3.0, 4.0)
    val v2 = Vector2D(1.0, 2.0)
    
    println("v1 = $v1 (magnitude = ${"%.2f".format(v1.magnitude)})")
    println("v2 = $v2")
    println()
    println("v1 + v2 = ${v1 + v2}")
    println("v1 - v2 = ${v1 - v2}")
    println("v1 * 2.0 = ${v1 * 2.0}")
    println("v1 / 2.0 = ${v1 / 2.0}")
    println("-v1 = ${-v1}")
    println("v1 normalized = ${v1.normalized()}")
    println("v1 · v2 = ${v1.dot(v2)}")
    
    val m1 = Matrix2x2(1.0, 2.0, 3.0, 4.0)
    val m2 = Matrix2x2(5.0, 6.0, 7.0, 8.0)
    
    println("\nm1 =\n$m1")
    println("\nm2 =\n$m2")
    println("\nm1 * m2 =\n${m1 * m2}")
    println("\ndet(m1) = ${m1.determinant}")
}
```

---

## ขั้นตอนที่ 59: Function Types และ First-Class Functions

```kotlin
// ============================================
// Function Types
// ============================================

// Type ของฟังก์ชัน: (ParameterTypes) -> ReturnType
val add: (Int, Int) -> Int = { a, b -> a + b }
val greet: (String) -> String = { name -> "Hello, $name!" }
val printLine: () -> Unit = { println("---") }
val double: (Int) -> Int = { it * 2 }  // ใช้ 'it' กับ single parameter

// ============================================
// Higher-Order Functions
// ============================================

fun applyOperation(a: Int, b: Int, operation: (Int, Int) -> Int): Int {
    return operation(a, b)
}

fun transformList(list: List<Int>, transform: (Int) -> Int): List<Int> {
    return list.map(transform)
}

// Return function
fun multiplier(factor: Int): (Int) -> Int {
    return { n -> n * factor }
}

fun main() {
    // เรียกใช้ function type
    println(add(3, 4))           // 7
    println(greet("Kotlin"))     // Hello, Kotlin!
    printLine()                  // ---
    println(double(5))           // 10
    
    // Higher-order functions
    println(applyOperation(10, 3, { a, b -> a + b }))  // 13
    println(applyOperation(10, 3, { a, b -> a * b }))  // 30
    println(applyOperation(10, 3, { a, b -> a - b }))  // 7
    
    // Trailing lambda syntax (lambda เป็น parameter สุดท้าย)
    println(applyOperation(10, 3) { a, b -> a / b })   // 3
    
    // Function reference
    println(applyOperation(10, 3, Int::plus))  // 13 (method reference)
    
    // Transform list
    val numbers = listOf(1, 2, 3, 4, 5)
    println(transformList(numbers) { it * it })  // [1, 4, 9, 16, 25]
    println(transformList(numbers, ::double))    // [2, 4, 6, 8, 10]
    
    // Return function
    val triple = multiplier(3)
    val quadruple = multiplier(4)
    
    println("\nMultipliers:")
    println("triple(5) = ${triple(5)}")      // 15
    println("quadruple(5) = ${quadruple(5)}") // 20
    
    // ใช้งานจริง
    val operations = mapOf(
        "+" to { a: Int, b: Int -> a + b },
        "-" to { a: Int, b: Int -> a - b },
        "*" to { a: Int, b: Int -> a * b },
        "/" to { a: Int, b: Int -> if (b != 0) a / b else 0 }
    )
    
    println("\nCalculator:")
    for ((op, func) in operations) {
        println("10 $op 3 = ${func(10, 3)}")
    }
}
```

---

## ขั้นตอนที่ 60: Scope Functions - let, run, with, apply, also

```kotlin
// ============================================
// Scope Functions - 5 functions สำคัญ
// ============================================

data class User(
    var name: String = "",
    var email: String = "",
    var age: Int = 0,
    var isActive: Boolean = false
)

fun main() {
    // ============================================
    // let - transform object และ return ค่าใหม่
    // Context: it | Returns: lambda result
    // ============================================
    
    val name: String? = "สมชาย"
    
    // ใช้บ่อยกับ nullable types
    val greeting = name?.let { 
        "สวัสดี, $it! ชื่อยาว ${it.length} ตัวอักษร"
    } ?: "ไม่ทราบชื่อ"
    println("let: $greeting")
    
    // Transform
    val numberString = "42"
    val doubled = numberString.let { 
        it.toInt() * 2  // return ค่าใหม่
    }
    println("let transform: $doubled")
    
    // ============================================
    // run - execute code block, return result
    // Context: this | Returns: lambda result
    // ============================================
    
    val user = User()
    val userSummary = user.run {
        name = "สมหญิง"
        email = "somying@example.com"
        age = 28
        isActive = true
        "User created: $name (${email})"  // return value
    }
    println("run: $userSummary")
    println("user after run: $user")
    
    // ============================================
    // with - คล้าย run แต่ส่ง object เป็น argument
    // Context: this | Returns: lambda result
    // ============================================
    
    val result = with(user) {
        // ไม่ต้อง user.xxx เพราะ this = user
        "Name: $name, Age: $age, Active: $isActive"
    }
    println("with: $result")
    
    // ============================================
    // apply - configure object, return same object
    // Context: this | Returns: object itself
    // ============================================
    
    val newUser = User().apply {
        name = "สมศักดิ์"
        email = "somsak@example.com"
        age = 35
        isActive = true
    }
    println("apply: $newUser")
    
    // Builder pattern ด้วย apply
    val stringBuilder = StringBuilder().apply {
        append("Hello")
        append(", ")
        append("Kotlin")
        append("!")
    }
    println("StringBuilder apply: $stringBuilder")
    
    // ============================================
    // also - side effects, return same object
    // Context: it | Returns: object itself
    // ============================================
    
    val processedUser = User().apply {
        name = "สมปอง"
        email = "sompong@example.com"
    }.also { user ->
        println("also - logging: Creating user ${user.name}")
        // Side effect เช่น logging, validation
    }
    println("also: $processedUser")
    
    // ============================================
    // เปรียบเทียบ Scope Functions
    // ============================================
    
    /*
    ┌──────────┬──────────┬──────────────┬─────────────────────────────┐
    │ Function │ Context  │ Return Value │ Use Case                     │
    ├──────────┼──────────┼──────────────┼─────────────────────────────┤
    │ let      │ it       │ Lambda result│ Null checks, transformation  │
    │ run      │ this     │ Lambda result│ Object init + compute result │
    │ with     │ this     │ Lambda result│ Grouping calls on an object  │
    │ apply    │ this     │ Object itself│ Object configuration         │
    │ also     │ it       │ Object itself│ Side effects (logging)       │
    └──────────┴──────────┴──────────────┴─────────────────────────────┘
    */
    
    // Chain scope functions
    val finalUser = User()
        .apply {
            name = "สมใจ"
            age = 22
        }
        .also { println("Created: ${it.name}") }
        .apply { isActive = true }
        .also { println("Activated: ${it.name}") }
    
    println("Final: $finalUser")
}
```

---

## แบบฝึกหัด Part 04

```kotlin
// แบบฝึกหัดที่ 1: ฟังก์ชัน filter ของ custom
fun <T> myFilter(list: List<T>, predicate: (T) -> Boolean): List<T> {
    // TODO: implement filter without using built-in filter
    return TODO()
}

// แบบฝึกหัดที่ 2: Pipeline function
// ทำ pipeline ของ operations
fun pipeline(vararg operations: (Int) -> Int): (Int) -> Int {
    // TODO: return function ที่ apply operations ทีละขั้น
    return TODO()
}

// แบบฝึกหัดที่ 3: Memoization
fun memoize(fn: (Int) -> Long): (Int) -> Long {
    // TODO: return function ที่ cache ผลลัพธ์
    return TODO()
}

fun main() {
    // ทดสอบ
    val numbers = listOf(1, 2, 3, 4, 5, 6, 7, 8, 9, 10)
    println(myFilter(numbers) { it % 2 == 0 })  // [2, 4, 6, 8, 10]
    println(myFilter(numbers) { it > 5 })         // [6, 7, 8, 9, 10]
    
    val transform = pipeline(
        { it * 2 },    // คูณ 2
        { it + 10 },   // บวก 10
        { it * it }    // ยกกำลัง 2
    )
    println(transform(3))  // ((3*2)+10)^2 = 256
    
    val slowFib: (Int) -> Long = { n ->
        if (n <= 1) n.toLong()
        else slowFib(n - 1) + slowFib(n - 2)  // recursive, slow
    }
    val fastFib = memoize(slowFib)
    println(fastFib(40))  // คำนวณเร็วกว่า
}
```

### เฉลย

```kotlin
fun <T> myFilter(list: List<T>, predicate: (T) -> Boolean): List<T> {
    val result = mutableListOf<T>()
    for (item in list) {
        if (predicate(item)) result.add(item)
    }
    return result
}

fun pipeline(vararg operations: (Int) -> Int): (Int) -> Int {
    return { input ->
        operations.fold(input) { acc, op -> op(acc) }
    }
}

fun memoize(fn: (Int) -> Long): (Int) -> Long {
    val cache = mutableMapOf<Int, Long>()
    return { n ->
        cache.getOrPut(n) { fn(n) }
    }
}
```

---

*Part 04 จบแล้ว | ก่อนหน้า: [Part 03](../part03/README.md) | ถัดไป: [Part 05](../part05/README.md)*
