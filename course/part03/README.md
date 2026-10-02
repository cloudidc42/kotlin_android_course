# Part 03: Control Flow - if, when, loops
## ขั้นตอนที่ 31-50

---

## ขั้นตอนที่ 31: if Expression

```kotlin
fun main() {
    // ============================================
    // if แบบ Statement (คุ้นเคยจาก Java)
    // ============================================
    
    val temperature = 35
    
    if (temperature > 30) {
        println("อากาศร้อนมาก!")
    } else if (temperature > 20) {
        println("อากาศอุ่น")
    } else if (temperature > 10) {
        println("อากาศเย็น")
    } else {
        println("อากาศหนาวมาก!")
    }
    
    // ============================================
    // if แบบ Expression (พิเศษใน Kotlin!)
    // ============================================
    
    // ใน Kotlin, if สามารถ return ค่าได้
    val max = if (10 > 20) 10 else 20
    println("max = $max")  // 20
    
    // แบบมีหลาย block
    val weatherMessage = if (temperature > 30) {
        val emoji = "🌡️"
        "$emoji ร้อนมาก $temperature°C"  // ค่าสุดท้ายคือ return value
    } else if (temperature > 20) {
        "อุ่นสบาย $temperature°C"
    } else {
        "เย็นสบาย $temperature°C"
    }
    println(weatherMessage)
    
    // ============================================
    // Single-line if (Ternary-like)
    // ============================================
    
    val score = 75
    val passed = if (score >= 60) "ผ่าน" else "ไม่ผ่าน"
    println("ผล: $passed")
    
    // ============================================
    // if ที่ซ้อนกัน (Nested if)
    // ============================================
    
    val age = 20
    val hasID = true
    
    if (age >= 18) {
        if (hasID) {
            println("เข้าได้")
        } else {
            println("ต้องมีบัตรประชาชน")
        }
    } else {
        println("อายุไม่ถึง 18 ปี")
    }
    
    // แบบกระชับกว่า
    val canEnter = age >= 18 && hasID
    println("เข้าได้: $canEnter")
}
```

---

## ขั้นตอนที่ 32: when Expression

```kotlin
fun main() {
    // ============================================
    // when - เหมือน switch แต่ทรงพลังกว่ามาก
    // ============================================
    
    val dayOfWeek = 3
    
    // แบบ Statement
    when (dayOfWeek) {
        1 -> println("วันจันทร์")
        2 -> println("วันอังคาร")
        3 -> println("วันพุธ")
        4 -> println("วันพฤหัสบดี")
        5 -> println("วันศุกร์")
        6 -> println("วันเสาร์")
        7 -> println("วันอาทิตย์")
        else -> println("ไม่ถูกต้อง")
    }
    
    // แบบ Expression (return ค่า)
    val dayName = when (dayOfWeek) {
        1 -> "จันทร์"
        2 -> "อังคาร"
        3 -> "พุธ"
        4 -> "พฤหัสบดี"
        5 -> "ศุกร์"
        6 -> "เสาร์"
        7 -> "อาทิตย์"
        else -> "ไม่ถูกต้อง"
    }
    println("วันที่ $dayOfWeek = $dayName")
    
    // ============================================
    // when ที่มีหลายค่าในกรณีเดียว
    // ============================================
    
    val isWeekend = when (dayOfWeek) {
        6, 7 -> true    // วันเสาร์ หรือ วันอาทิตย์
        else -> false
    }
    println("เป็นวันหยุด: $isWeekend")
    
    // ============================================
    // when กับ Range
    // ============================================
    
    val score = 85
    val grade = when (score) {
        in 90..100 -> "A"
        in 80..89  -> "B"
        in 70..79  -> "C"
        in 60..69  -> "D"
        in 0..59   -> "F"
        else       -> "คะแนนไม่ถูกต้อง"
    }
    println("คะแนน $score = เกรด $grade")
    
    // ============================================
    // when กับ String
    // ============================================
    
    val command = "start"
    when (command) {
        "start", "begin", "go" -> println("เริ่มต้น!")
        "stop", "end", "quit"  -> println("หยุด!")
        "pause"                 -> println("พัก...")
        else                    -> println("คำสั่งไม่รู้จัก: $command")
    }
    
    // ============================================
    // when ไม่ต้องมี argument (แบบ if-else chain)
    // ============================================
    
    val temperature = 35
    val humidity = 80
    
    val weather = when {
        temperature > 35 && humidity > 80 -> "ร้อนอบอ้าว"
        temperature > 30                   -> "ร้อน"
        temperature > 20                   -> "อุ่น"
        temperature > 10                   -> "เย็น"
        else                               -> "หนาว"
    }
    println("สภาพอากาศ: $weather")
    
    // ============================================
    // when กับ Type Checking
    // ============================================
    
    fun describeValue(value: Any): String = when (value) {
        is Int     -> "เลขจำนวนเต็ม: $value"
        is Double  -> "เลขทศนิยม: $value"
        is String  -> "ข้อความยาว ${value.length} ตัวอักษร: $value"
        is Boolean -> "ค่าตรรกะ: $value"
        is List<*> -> "List มี ${value.size} รายการ"
        null       -> "null"
        else       -> "ชนิดไม่รู้จัก: ${value::class.simpleName}"
    }
    
    println("\nType checking with when:")
    println(describeValue(42))
    println(describeValue(3.14))
    println(describeValue("Hello"))
    println(describeValue(true))
    println(describeValue(listOf(1, 2, 3)))
    println(describeValue(null))
}
```

---

## ขั้นตอนที่ 33: for Loop

```kotlin
fun main() {
    // ============================================
    // for loop กับ Range
    // ============================================
    
    // 1 ถึง 5
    print("1 ถึง 5: ")
    for (i in 1..5) {
        print("$i ")
    }
    println()
    
    // 1 ถึง 4 (ไม่รวม 5)
    print("1 ถึง 4: ")
    for (i in 1 until 5) {
        print("$i ")
    }
    println()
    
    // นับถอยหลัง
    print("นับถอยหลัง: ")
    for (i in 5 downTo 1) {
        print("$i ")
    }
    println()
    
    // Step (กระโดดทีละ 2)
    print("เลขคู่: ")
    for (i in 2..10 step 2) {
        print("$i ")
    }
    println()
    
    // ============================================
    // for loop กับ Collection
    // ============================================
    
    val fruits = listOf("มะม่วง", "กล้วย", "ส้ม", "แอปเปิ้ล")
    
    // แบบธรรมดา
    println("\nผลไม้:")
    for (fruit in fruits) {
        println("  - $fruit")
    }
    
    // แบบมี index
    println("\nผลไม้พร้อม index:")
    for ((index, fruit) in fruits.withIndex()) {
        println("  ${index + 1}. $fruit")
    }
    
    // forEachIndexed (functional style)
    println("\nforEachIndexed:")
    fruits.forEachIndexed { index, fruit ->
        println("  [${index}] $fruit")
    }
    
    // ============================================
    // for loop กับ Map
    // ============================================
    
    val scores = mapOf(
        "สมชาย" to 90,
        "สมหญิง" to 85,
        "สมศักดิ์" to 78
    )
    
    println("\nคะแนนสอบ:")
    for ((name, score) in scores) {
        println("  $name: $score คะแนน")
    }
    
    // ============================================
    // for loop กับ String
    // ============================================
    
    val text = "Hello"
    print("\nตัวอักษรใน '$text': ")
    for (char in text) {
        print("$char ")
    }
    println()
    
    // ============================================
    // Nested for loop
    // ============================================
    
    println("\nตารางสูตรคูณ 1-5:")
    for (i in 1..5) {
        for (j in 1..5) {
            print("${(i * j).toString().padStart(3)}")
        }
        println()
    }
}
```

---

## ขั้นตอนที่ 34: while และ do-while Loop

```kotlin
fun main() {
    // ============================================
    // while loop
    // ============================================
    
    var count = 1
    print("while 1-5: ")
    while (count <= 5) {
        print("$count ")
        count++
    }
    println()
    
    // ============================================
    // do-while loop
    // ============================================
    
    // do-while ทำงานอย่างน้อย 1 ครั้งก่อนตรวจเงื่อนไข
    var num = 10
    print("do-while (ทำแม้ condition false ตั้งแต่แรก): ")
    do {
        print("$num ")
        num--
    } while (num > 10)  // condition เป็น false แต่ก็ทำ 1 ครั้ง
    println()
    
    // ============================================
    // while กับ User Input Simulation
    // ============================================
    
    // จำลองการรับ input จาก user
    val inputs = listOf("ไม่ออก", "ไม่ออก", "ออก")
    var inputIndex = 0
    var userInput: String
    
    println("\nจำลอง input loop:")
    do {
        userInput = inputs[inputIndex++]
        println("User input: $userInput")
    } while (userInput != "ออก")
    println("ออกจากโปรแกรมแล้ว")
    
    // ============================================
    // while ที่อาจเป็น Infinite Loop (ระวัง!)
    // ============================================
    
    // ตัวอย่างที่ปลอดภัย - มีเงื่อนไขออก
    var fibonacci1 = 0
    var fibonacci2 = 1
    print("\nFibonacci < 100: ")
    while (fibonacci1 < 100) {
        print("$fibonacci1 ")
        val next = fibonacci1 + fibonacci2
        fibonacci1 = fibonacci2
        fibonacci2 = next
    }
    println()
}
```

---

## ขั้นตอนที่ 35: break, continue และ Labels

```kotlin
fun main() {
    // ============================================
    // break - ออกจาก loop
    // ============================================
    
    println("break example:")
    for (i in 1..10) {
        if (i == 6) break  // หยุดเมื่อเจอ 6
        print("$i ")
    }
    println()  // 1 2 3 4 5
    
    // ============================================
    // continue - ข้ามไป iteration ถัดไป
    // ============================================
    
    println("\ncontinue (ข้ามเลขคู่):")
    for (i in 1..10) {
        if (i % 2 == 0) continue  // ข้ามเลขคู่
        print("$i ")
    }
    println()  // 1 3 5 7 9
    
    // ============================================
    // Labels - สำหรับ Nested Loops
    // ============================================
    
    // ปัญหา: break ใน nested loop จะออกแค่ loop ในสุด
    println("\nbreak without label (ออกแค่ loop ใน):")
    for (i in 1..3) {
        for (j in 1..3) {
            if (j == 2) break  // ออกแค่ inner loop
            print("($i,$j) ")
        }
    }
    println()
    // (1,1) (2,1) (3,1)
    
    // แก้ด้วย Label
    println("\nbreak with label (ออก outer loop):")
    outer@ for (i in 1..3) {
        for (j in 1..3) {
            if (j == 2) break@outer  // ออก outer loop
            print("($i,$j) ")
        }
    }
    println()
    // (1,1)
    
    // continue with label
    println("\ncontinue with label:")
    outer@ for (i in 1..3) {
        for (j in 1..3) {
            if (j == 2) continue@outer  // ไป iteration ถัดไปของ outer loop
            print("($i,$j) ")
        }
    }
    println()
    // (1,1) (2,1) (3,1)
    
    // ============================================
    // Return from Lambda กับ Labels
    // ============================================
    
    val numbers = listOf(1, 2, 3, 4, 5, 6, 7, 8, 9, 10)
    
    // return ธรรมดาจะออกจากฟังก์ชัน main ทั้งหมด!
    // ถ้าต้องการออกแค่ lambda ต้องใช้ label
    
    println("\nreturn@forEach (ข้ามเลขคี่):")
    numbers.forEach { num ->
        if (num % 2 != 0) return@forEach  // ออกแค่ lambda iteration นี้
        print("$num ")
    }
    println()
    // 2 4 6 8 10
}
```

---

## ขั้นตอนที่ 36: repeat และ Iteration Functions

```kotlin
fun main() {
    // ============================================
    // repeat - ทำซ้ำ n ครั้ง
    // ============================================
    
    print("repeat 5 ครั้ง: ")
    repeat(5) { index ->
        print("[$index] ")
    }
    println()  // [0] [1] [2] [3] [4]
    
    // ============================================
    // forEach
    // ============================================
    
    val numbers = listOf(1, 2, 3, 4, 5)
    
    println("\nforEach:")
    numbers.forEach { print("$it ") }
    println()
    
    numbers.forEachIndexed { index, value ->
        print("[$index]=$value ")
    }
    println()
    
    // ============================================
    // Iteration ใน Map
    // ============================================
    
    val capitals = mapOf(
        "ไทย" to "กรุงเทพฯ",
        "ญี่ปุ่น" to "โตเกียว",
        "อังกฤษ" to "ลอนดอน"
    )
    
    println("\nเมืองหลวง:")
    capitals.forEach { (country, capital) ->
        println("  $country -> $capital")
    }
    
    // ============================================
    // generateSequence - สร้าง Infinite Sequence
    // ============================================
    
    // Fibonacci sequence
    val fibSequence = generateSequence(Pair(0, 1)) { (a, b) ->
        Pair(b, a + b)
    }.map { it.first }
    
    print("\nFibonacci แรก 10 ตัว: ")
    fibSequence.take(10).forEach { print("$it ") }
    println()
    // 0 1 1 2 3 5 8 13 21 34
    
    // Power of 2
    val powersOf2 = generateSequence(1) { it * 2 }
    print("Powers of 2 (< 1000): ")
    powersOf2.takeWhile { it < 1000 }.forEach { print("$it ") }
    println()
    // 1 2 4 8 16 32 64 128 256 512
    
    // ============================================
    // Iterator - การวนซ้ำแบบ Manual
    // ============================================
    
    val list = listOf("A", "B", "C", "D")
    val iterator = list.iterator()
    
    println("\nManual iteration:")
    while (iterator.hasNext()) {
        val item = iterator.next()
        print("$item ")
    }
    println()
}
```

---

## ขั้นตอนที่ 37: Pattern Matching กับ when

```kotlin
// Sealed class สำหรับตัวอย่าง (จะเรียนละเอียดใน Part 12)
sealed class Shape {
    data class Circle(val radius: Double) : Shape()
    data class Rectangle(val width: Double, val height: Double) : Shape()
    data class Triangle(val base: Double, val height: Double) : Shape()
}

fun calculateArea(shape: Shape): Double = when (shape) {
    is Shape.Circle    -> Math.PI * shape.radius * shape.radius
    is Shape.Rectangle -> shape.width * shape.height
    is Shape.Triangle  -> 0.5 * shape.base * shape.height
}

fun describeShape(shape: Shape): String = when (shape) {
    is Shape.Circle -> "วงกลมรัศมี ${shape.radius}"
    is Shape.Rectangle -> "สี่เหลี่ยม ${shape.width}×${shape.height}"
    is Shape.Triangle -> "สามเหลี่ยมฐาน ${shape.base} สูง ${shape.height}"
}

fun main() {
    val shapes = listOf(
        Shape.Circle(5.0),
        Shape.Rectangle(4.0, 6.0),
        Shape.Triangle(3.0, 4.0)
    )
    
    println("รูปทรงและพื้นที่:")
    shapes.forEach { shape ->
        val description = describeShape(shape)
        val area = calculateArea(shape)
        println("  $description -> พื้นที่ = %.2f".format(area))
    }
    
    // ============================================
    // Destructuring กับ when
    // ============================================
    
    data class Point(val x: Int, val y: Int)
    
    fun quadrant(point: Point): String {
        val (x, y) = point  // Destructuring
        return when {
            x > 0 && y > 0 -> "Quadrant I"
            x < 0 && y > 0 -> "Quadrant II"
            x < 0 && y < 0 -> "Quadrant III"
            x > 0 && y < 0 -> "Quadrant IV"
            x == 0 && y == 0 -> "Origin"
            x == 0 -> "Y-axis"
            else -> "X-axis"
        }
    }
    
    val points = listOf(
        Point(3, 4), Point(-2, 5), Point(-1, -3),
        Point(4, -2), Point(0, 0), Point(0, 5)
    )
    
    println("\nQuadrant:")
    points.forEach { point ->
        println("  $point -> ${quadrant(point)}")
    }
}
```

---

## ขั้นตอนที่ 38: Guard Conditions และ Early Return

```kotlin
// ============================================
// Early Return Pattern - ออกจากฟังก์ชันเร็วๆ
// ============================================

fun validateEmail(email: String?): String {
    // Guard: ตรวจสอบ null ก่อน
    if (email == null) return "Email ต้องไม่เป็น null"
    
    // Guard: ตรวจสอบว่าไม่ว่าง
    if (email.isBlank()) return "Email ต้องไม่ว่าง"
    
    // Guard: ตรวจสอบรูปแบบ
    if (!email.contains("@")) return "Email ต้องมี @"
    
    val parts = email.split("@")
    if (parts.size != 2) return "รูปแบบ Email ไม่ถูกต้อง"
    
    val (local, domain) = parts
    if (local.isEmpty()) return "ส่วนหน้า @ ต้องไม่ว่าง"
    if (!domain.contains(".")) return "Domain ต้องมี ."
    
    // ผ่านทุก guard แล้ว
    return "Email ถูกต้อง: $email"
}

fun divideNumbers(a: Double, b: Double): Double? {
    if (b == 0.0) return null  // Guard: ห้ามหารด้วย 0
    return a / b
}

fun processAge(age: Int): String {
    if (age < 0) return "อายุต้องไม่ติดลบ"
    if (age > 150) return "อายุไม่สมเหตุสมผล"
    
    return when (age) {
        0 -> "แรกเกิด"
        in 1..12 -> "เด็ก"
        in 13..17 -> "วัยรุ่น"
        in 18..59 -> "ผู้ใหญ่"
        else -> "ผู้สูงอายุ"
    }
}

fun main() {
    // ทดสอบ validateEmail
    println("Email Validation:")
    listOf(null, "", "notanemail", "a@", "@b", "a@b", "user@example.com").forEach { email ->
        println("  \"$email\" -> ${validateEmail(email)}")
    }
    
    // ทดสอบ divideNumbers
    println("\nDivision:")
    println("  10 / 2 = ${divideNumbers(10.0, 2.0)}")
    println("  10 / 0 = ${divideNumbers(10.0, 0.0)}")
    
    // ทดสอบ processAge
    println("\nAge Processing:")
    listOf(-5, 0, 5, 15, 25, 65, 200).forEach { age ->
        println("  Age $age: ${processAge(age)}")
    }
}
```

---

## ขั้นตอนที่ 39: Conditional Expressions สำหรับ Android

```kotlin
// ตัวอย่างการใช้ if/when ใน Android Context

// Simulating Android-like code
enum class NetworkStatus { CONNECTED, DISCONNECTED, LOADING }
enum class UserRole { ADMIN, USER, GUEST }

data class User(val name: String, val role: UserRole, val isLoggedIn: Boolean)

fun getWelcomeMessage(user: User?): String = when {
    user == null -> "กรุณาเข้าสู่ระบบ"
    !user.isLoggedIn -> "กรุณาเข้าสู่ระบบ, ${user.name}"
    user.role == UserRole.ADMIN -> "ยินดีต้อนรับ Admin: ${user.name}"
    user.role == UserRole.USER -> "สวัสดี, ${user.name}"
    else -> "ยินดีต้อนรับ, แขก"
}

fun getNetworkMessage(status: NetworkStatus): String = when (status) {
    NetworkStatus.CONNECTED    -> "เชื่อมต่อแล้ว ✅"
    NetworkStatus.DISCONNECTED -> "ไม่มีการเชื่อมต่อ ❌"
    NetworkStatus.LOADING      -> "กำลังเชื่อมต่อ... ⏳"
}

fun canAccessFeature(user: User?, feature: String): Boolean {
    if (user == null || !user.isLoggedIn) return false
    
    return when (feature) {
        "dashboard"   -> true  // ทุกคนเข้าได้
        "settings"    -> user.role == UserRole.ADMIN || user.role == UserRole.USER
        "admin_panel" -> user.role == UserRole.ADMIN
        "reports"     -> user.role in listOf(UserRole.ADMIN, UserRole.USER)
        else          -> false
    }
}

fun main() {
    val adminUser = User("สมชาย", UserRole.ADMIN, true)
    val regularUser = User("สมหญิง", UserRole.USER, true)
    val guestUser = User("แขก", UserRole.GUEST, false)
    
    println("Welcome Messages:")
    println(getWelcomeMessage(null))
    println(getWelcomeMessage(adminUser))
    println(getWelcomeMessage(regularUser))
    println(getWelcomeMessage(guestUser))
    
    println("\nNetwork Status:")
    NetworkStatus.values().forEach { status ->
        println(getNetworkMessage(status))
    }
    
    val features = listOf("dashboard", "settings", "admin_panel", "reports")
    println("\nFeature Access:")
    listOf(adminUser, regularUser).forEach { user ->
        println("\n${user.name} (${user.role}):")
        features.forEach { feature ->
            val canAccess = canAccessFeature(user, feature)
            val icon = if (canAccess) "✅" else "❌"
            println("  $icon $feature")
        }
    }
}
```

---

## ขั้นตอนที่ 40: Loop Performance และ Best Practices

```kotlin
fun main() {
    // ============================================
    // เปรียบเทียบ Performance ของ loops
    // ============================================
    
    val size = 1_000_000
    val list = (1..size).toList()
    
    // วัดเวลา helper function
    fun measureTime(name: String, block: () -> Unit) {
        val start = System.nanoTime()
        block()
        val end = System.nanoTime()
        println("$name: ${(end - start) / 1_000_000}ms")
    }
    
    // for loop
    measureTime("for loop") {
        var sum = 0L
        for (n in list) {
            sum += n
        }
    }
    
    // forEach
    measureTime("forEach") {
        var sum = 0L
        list.forEach { sum += it }
    }
    
    // forEachIndexed (ช้ากว่าเล็กน้อย)
    measureTime("forEachIndexed") {
        var sum = 0L
        list.forEachIndexed { _, n -> sum += n }
    }
    
    // sum() ใช้ built-in (เร็วที่สุด)
    measureTime("sum()") {
        list.sum()
    }
    
    // ============================================
    // Best Practices
    // ============================================
    
    // 1. ใช้ functional operations แทน loops เมื่อเป็นไปได้
    val numbers = (1..100).toList()
    
    // ไม่ดี - ใช้ loop
    val evensBad = mutableListOf<Int>()
    for (n in numbers) {
        if (n % 2 == 0) evensBad.add(n)
    }
    
    // ดี - functional
    val evensGood = numbers.filter { it % 2 == 0 }
    
    println("\nEven numbers (first 5): ${evensGood.take(5)}")
    
    // 2. ใช้ sequence สำหรับ large collections
    val result = (1..1_000_000)
        .asSequence()  // lazy evaluation
        .filter { it % 2 == 0 }
        .map { it * 3 }
        .take(5)
        .toList()
    
    println("Sequence result: $result")
    
    // 3. ใช้ indices เมื่อต้องการ index
    val items = listOf("A", "B", "C")
    
    // แบบไม่ดี
    for (i in 0 until items.size) {
        println("$i: ${items[i]}")
    }
    
    // แบบดีกว่า
    for ((i, item) in items.withIndex()) {
        println("$i: $item")
    }
    
    // แบบดีที่สุด (functional)
    items.forEachIndexed { i, item -> println("$i: $item") }
}
```

---

## สรุปและแบบฝึกหัด Part 03

### Workshop: เกม Guess the Number

```kotlin
import kotlin.random.Random

fun main() {
    val secretNumber = Random.nextInt(1, 101)  // 1-100
    var attempts = 0
    val maxAttempts = 7
    
    println("=== เกมทายตัวเลข ===")
    println("ฉันคิดเลขระหว่าง 1-100 ไว้ในใจ")
    println("คุณมี $maxAttempts ครั้งในการทาย")
    
    // จำลอง guesses
    val guesses = listOf(50, 75, 62, 68, 65, 67, 66)
    
    var won = false
    for (guess in guesses) {
        if (attempts >= maxAttempts) break
        
        attempts++
        println("\nครั้งที่ $attempts: ทาย $guess")
        
        when {
            guess == secretNumber -> {
                println("🎉 ถูกต้อง! ตัวเลขคือ $secretNumber")
                println("คุณทายถูกใน $attempts ครั้ง")
                won = true
                break
            }
            guess < secretNumber -> println("มากกว่า $guess ⬆️")
            else                  -> println("น้อยกว่า $guess ⬇️")
        }
    }
    
    if (!won) {
        println("\n😢 หมดจำนวนครั้งแล้ว! ตัวเลขคือ $secretNumber")
    }
}
```

### แบบฝึกหัด

```kotlin
// แบบฝึกหัดที่ 1: FizzBuzz
// - หาร 3 ลงตัว พิมพ์ "Fizz"
// - หาร 5 ลงตัว พิมพ์ "Buzz"
// - หาร 15 ลงตัว พิมพ์ "FizzBuzz"
// - อื่นๆ พิมพ์ตัวเลขนั้น

fun fizzBuzz(n: Int) {
    for (i in 1..n) {
        // TODO: เติมโค้ด
    }
}

// แบบฝึกหัดที่ 2: หาจำนวนเฉพาะ (Prime Numbers)
fun isPrime(n: Int): Boolean {
    // TODO: return true ถ้า n เป็นจำนวนเฉพาะ
    return TODO()
}

fun getPrimes(upTo: Int): List<Int> {
    // TODO: return list ของจำนวนเฉพาะทั้งหมดที่ <= upTo
    return TODO()
}

// แบบฝึกหัดที่ 3: Pyramid
fun printPyramid(height: Int) {
    // TODO: พิมพ์พีระมิดสูง height ชั้น
    // ตัวอย่าง height = 4:
    //    *
    //   ***
    //  *****
    // *******
}

fun main() {
    fizzBuzz(20)
    println("\nPrimes up to 50: ${getPrimes(50)}")
    printPyramid(5)
}
```

### เฉลย

```kotlin
// เฉลย FizzBuzz
fun fizzBuzz(n: Int) {
    for (i in 1..n) {
        println(when {
            i % 15 == 0 -> "FizzBuzz"
            i % 3 == 0  -> "Fizz"
            i % 5 == 0  -> "Buzz"
            else         -> "$i"
        })
    }
}

// เฉลย isPrime
fun isPrime(n: Int): Boolean {
    if (n < 2) return false
    if (n == 2) return true
    if (n % 2 == 0) return false
    for (i in 3..Math.sqrt(n.toDouble()).toInt() step 2) {
        if (n % i == 0) return false
    }
    return true
}

fun getPrimes(upTo: Int): List<Int> = (2..upTo).filter { isPrime(it) }

// เฉลย Pyramid
fun printPyramid(height: Int) {
    for (i in 1..height) {
        val spaces = " ".repeat(height - i)
        val stars = "*".repeat(2 * i - 1)
        println("$spaces$stars")
    }
}
```

---

*Part 03 จบแล้ว | ก่อนหน้า: [Part 02](../part02/README.md) | ถัดไป: [Part 04](../part04/README.md)*
