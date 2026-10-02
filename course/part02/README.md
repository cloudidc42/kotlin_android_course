# Part 02: ตัวแปร ชนิดข้อมูล และ Operators
## ขั้นตอนที่ 11-30

---

## ขั้นตอนที่ 11: ชนิดข้อมูลพื้นฐาน (Primitive Types)

```kotlin
fun main() {
    // ============================================
    // ตัวเลขจำนวนเต็ม (Integer Types)
    // ============================================
    
    val byteVal: Byte = 127           // -128 ถึง 127 (8-bit)
    val shortVal: Short = 32767       // -32768 ถึง 32767 (16-bit)
    val intVal: Int = 2147483647      // -2^31 ถึง 2^31-1 (32-bit)
    val longVal: Long = 9223372036854775807L  // 64-bit (ต้องมี L ต่อท้าย)
    
    println("Byte: $byteVal")
    println("Short: $shortVal")
    println("Int: $intVal")
    println("Long: $longVal")
    
    // ขนาดหน่วยความจำ
    println("\nขนาดหน่วยความจำ:")
    println("Byte: ${Byte.SIZE_BYTES} bytes = ${Byte.SIZE_BITS} bits")
    println("Short: ${Short.SIZE_BYTES} bytes = ${Short.SIZE_BITS} bits")
    println("Int: ${Int.SIZE_BYTES} bytes = ${Int.SIZE_BITS} bits")
    println("Long: ${Long.SIZE_BYTES} bytes = ${Long.SIZE_BITS} bits")
    
    // ค่าสูงสุดและต่ำสุด
    println("\nค่าสูงสุด/ต่ำสุด:")
    println("Int.MAX_VALUE = ${Int.MAX_VALUE}")
    println("Int.MIN_VALUE = ${Int.MIN_VALUE}")
    println("Long.MAX_VALUE = ${Long.MAX_VALUE}")
}
```

```kotlin
fun main() {
    // ============================================
    // ตัวเลขทศนิยม (Floating Point Types)
    // ============================================
    
    val floatVal: Float = 3.14f       // 32-bit (ต้องมี f ต่อท้าย)
    val doubleVal: Double = 3.14159265358979  // 64-bit (ความแม่นยำสูงกว่า)
    
    println("Float: $floatVal")
    println("Double: $doubleVal")
    
    // ความแตกต่างของความแม่นยำ
    val pi_float: Float = 3.141592653589793f
    val pi_double: Double = 3.141592653589793
    
    println("\nความแม่นยำ:")
    println("Float Pi: $pi_float")    // 3.1415927 (ตัดที่ 7 ตำแหน่ง)
    println("Double Pi: $pi_double")  // 3.141592653589793 (15+ ตำแหน่ง)
    
    // Scientific Notation
    val avogadro = 6.022e23   // 6.022 × 10^23
    val electron = 1.6e-19    // 1.6 × 10^-19
    println("\nAvogadro's number: $avogadro")
    println("Electron charge: $electron")
    
    // Special values
    println("\nSpecial Values:")
    println("Double.NaN = ${Double.NaN}")
    println("Double.POSITIVE_INFINITY = ${Double.POSITIVE_INFINITY}")
    println("Double.NEGATIVE_INFINITY = ${Double.NEGATIVE_INFINITY}")
    println("1.0 / 0.0 = ${1.0 / 0.0}")  // Infinity
    println("0.0 / 0.0 = ${0.0 / 0.0}")  // NaN
}
```

---

## ขั้นตอนที่ 12: Boolean และ Char

```kotlin
fun main() {
    // ============================================
    // Boolean
    // ============================================
    
    val isKotlinFun: Boolean = true
    val isJavaFun: Boolean = false
    
    println("Kotlin is fun: $isKotlinFun")
    println("Java is fun: $isJavaFun")
    
    // Boolean Operations
    println("\nBoolean Operations:")
    println("true AND true = ${true && true}")    // true
    println("true AND false = ${true && false}")  // false
    println("true OR false = ${true || false}")   // true
    println("NOT true = ${!true}")                // false
    println("NOT false = ${!false}")              // true
    
    // Short-circuit evaluation
    var count = 0
    val result1 = (true || ++count > 0)   // count ไม่เพิ่ม เพราะ true ทำให้หยุดทันที
    println("\nShort-circuit OR: count = $count")  // 0
    
    val result2 = (false && ++count > 0)  // count ไม่เพิ่ม เพราะ false ทำให้หยุดทันที
    println("Short-circuit AND: count = $count")   // 0
    
    // ============================================
    // Char
    // ============================================
    
    val letterA: Char = 'A'
    val letterZ: Char = 'Z'
    val thaiKo: Char = 'ก'
    val digitFive: Char = '5'
    val newLine: Char = '\n'
    val tab: Char = '\t'
    val unicode: Char = 'A'  // 'A' ใน Unicode
    
    println("\nChar Examples:")
    println("Letter A: $letterA")
    println("Thai Ko: $thaiKo")
    println("Unicode A: $unicode")
    
    // Char to Int (ASCII/Unicode value)
    println("\nChar to Int:")
    println("'A'.code = ${'A'.code}")  // 65
    println("'a'.code = ${'a'.code}")  // 97
    println("'ก'.code = ${'ก'.code}")  // 3585
    
    // Int to Char
    println("\nInt to Char:")
    println("65.toChar() = ${65.toChar()}")    // A
    println("3585.toChar() = ${3585.toChar()}") // ก
    
    // Char operations
    val nextChar = letterA + 1  // B
    println("\nA + 1 = $nextChar")
    println("Z - A = ${letterZ - letterA}")  // 25
    
    // ตรวจสอบ Char
    println("\nChar checking:")
    println("'A'.isLetter() = ${'A'.isLetter()}")    // true
    println("'5'.isDigit() = ${'5'.isDigit()}")       // true
    println("' '.isWhitespace() = ${' '.isWhitespace()}")  // true
    println("'A'.isUpperCase() = ${'A'.isUpperCase()}")    // true
    println("'a'.isLowerCase() = ${'a'.isLowerCase()}")    // true
}
```

---

## ขั้นตอนที่ 13: String และ String Operations

```kotlin
fun main() {
    // ============================================
    // String พื้นฐาน
    // ============================================
    
    val name = "Kotlin"
    val message = "Hello, World!"
    val emptyString = ""
    val multiLine = """
        บรรทัดที่ 1
        บรรทัดที่ 2
        บรรทัดที่ 3
    """.trimIndent()
    
    println(multiLine)
    
    // String Properties
    println("\nString Properties:")
    println("ความยาว: ${name.length}")      // 6
    println("ว่างหรือไม่: ${emptyString.isEmpty()}")  // true
    println("ไม่ว่าง: ${name.isNotEmpty()}")         // true
    
    // ============================================
    // String Template (การแทรกตัวแปรใน String)
    // ============================================
    
    val firstName = "สมชาย"
    val lastName = "ใจดี"
    val age = 25
    val salary = 50000.0
    
    // แบบง่าย
    println("\nString Templates:")
    println("สวัสดี $firstName")
    
    // แบบ expression
    println("ชื่อเต็ม: ${firstName + " " + lastName}")
    println("อายุปีหน้า: ${age + 1}")
    println("เงินเดือนหลังหักภาษี: ${salary * 0.9}")
    
    // Format number
    println("เงินเดือน: %.2f บาท".format(salary))
    
    // ============================================
    // String Functions ที่ใช้บ่อย
    // ============================================
    
    val text = "  Hello, Kotlin!  "
    
    println("\nString Functions:")
    println("uppercase: ${text.uppercase()}")
    println("lowercase: ${text.lowercase()}")
    println("trim: '${text.trim()}'")
    println("trimStart: '${text.trimStart()}'")
    println("trimEnd: '${text.trimEnd()}'")
    println("replace: ${text.replace("Hello", "Hi")}")
    println("contains: ${text.contains("Kotlin")}")
    println("startsWith: ${text.trim().startsWith("Hello")}")
    println("endsWith: ${text.trim().endsWith("!")}")
    
    // ============================================
    // String Indexing และ Substring
    // ============================================
    
    val str = "Hello, Kotlin!"
    
    println("\nIndexing:")
    println("str[0] = ${str[0]}")        // H
    println("str[7] = ${str[7]}")        // K
    println("str.last() = ${str.last()}") // !
    println("str.first() = ${str.first()}") // H
    
    println("\nSubstring:")
    println("str.substring(7) = ${str.substring(7)}")        // Kotlin!
    println("str.substring(7, 13) = ${str.substring(7, 13)}") // Kotlin
    println("str.take(5) = ${str.take(5)}")                  // Hello
    println("str.drop(7) = ${str.drop(7)}")                  // Kotlin!
    println("str.takeLast(7) = ${str.takeLast(7)}")          // Kotlin!
    println("str.dropLast(1) = ${str.dropLast(1)}")          // Hello, Kotlin
    
    // ============================================
    // String Split และ Join
    // ============================================
    
    val csv = "มะม่วง,กล้วย,ส้ม,แอปเปิ้ล"
    val fruits = csv.split(",")
    println("\nSplit: $fruits")
    
    val joined = fruits.joinToString(" | ")
    println("Join: $joined")
    
    val joinedWithBrackets = fruits.joinToString(
        separator = ", ",
        prefix = "[",
        postfix = "]"
    )
    println("Join with brackets: $joinedWithBrackets")
}
```

---

## ขั้นตอนที่ 14: Type Inference (การอนุมานชนิดข้อมูล)

```kotlin
fun main() {
    // ============================================
    // Type Inference - Kotlin อนุมานชนิดอัตโนมัติ
    // ============================================
    
    // ไม่ต้องระบุชนิดข้อมูล Kotlin รู้เองจากค่าที่กำหนด
    val name = "Kotlin"              // String
    val age = 25                     // Int
    val salary = 50000.0             // Double
    val isActive = true              // Boolean
    val pi = 3.14159f                // Float (มี f ต่อท้าย)
    val bigNumber = 100L             // Long (มี L ต่อท้าย)
    
    // ตรวจสอบชนิดข้อมูล
    println("name is ${name::class.simpleName}")       // String
    println("age is ${age::class.simpleName}")         // Int
    println("salary is ${salary::class.simpleName}")   // Double
    println("pi is ${pi::class.simpleName}")           // Float
    println("bigNumber is ${bigNumber::class.simpleName}") // Long
    
    // ============================================
    // การระบุชนิดข้อมูลอย่างชัดเจน (Explicit Type)
    // ============================================
    
    val x: Int = 42
    val y: Double = 42.0  // ต้องระบุ .0 ไม่งั้น Kotlin จะคิดว่าเป็น Int
    val z: String = "42"
    
    // เมื่อไรควรระบุชนิดข้อมูล:
    // 1. เมื่อต้องการชนิดที่แตกต่างจากที่ Kotlin อนุมาน
    val shortNum: Short = 100  // ถ้าไม่ระบุ Kotlin จะเป็น Int
    val floatNum: Float = 3.14f  // ถ้าไม่มี f ต้องระบุ Float
    
    // 2. เมื่อประกาศตัวแปรโดยไม่กำหนดค่า
    val lateValue: String  // ต้องระบุชนิด
    lateValue = "assigned later"
    println(lateValue)
    
    // 3. เพื่อความชัดเจนของโค้ด
    val result: Boolean = checkAge(age)
    println("ผ่านเงื่อนไขอายุ: $result")
}

fun checkAge(age: Int): Boolean = age >= 18
```

---

## ขั้นตอนที่ 15: Type Conversion (การแปลงชนิดข้อมูล)

```kotlin
fun main() {
    // ============================================
    // Explicit Type Conversion
    // Kotlin ไม่มี Implicit Conversion ต้องแปลงเอง
    // ============================================
    
    val intNum = 42
    
    // Int -> other types
    val toByte: Byte = intNum.toByte()
    val toShort: Short = intNum.toShort()
    val toLong: Long = intNum.toLong()
    val toFloat: Float = intNum.toFloat()
    val toDouble: Double = intNum.toDouble()
    val toChar: Char = intNum.toChar()  // '*' (ASCII 42)
    val toString: String = intNum.toString()
    
    println("Int $intNum แปลงเป็น:")
    println("  Byte: $toByte")
    println("  Short: $toShort")
    println("  Long: $toLong")
    println("  Float: $toFloat")
    println("  Double: $toDouble")
    println("  Char: $toChar")
    println("  String: $toString")
    
    // ============================================
    // Numeric Conversion อันตราย!
    // ============================================
    
    val bigInt = 300
    val toByte2: Byte = bigInt.toByte()  // Overflow!
    println("\n300.toByte() = $toByte2")  // 44 (wraps around)
    
    val doubleNum = 3.99
    val toInt: Int = doubleNum.toInt()  // ตัดทิ้ง (ไม่ปัดเศษ)
    println("3.99.toInt() = $toInt")  // 3
    
    val roundedInt = Math.round(doubleNum).toInt()
    println("3.99 rounded = $roundedInt")  // 4
    
    // ============================================
    // String Conversion
    // ============================================
    
    // String to Number
    val strInt = "123"
    val strDouble = "45.67"
    val strInvalid = "abc"
    
    val parsedInt = strInt.toInt()
    val parsedDouble = strDouble.toDouble()
    
    println("\nString to Number:")
    println("\"123\".toInt() = $parsedInt")
    println("\"45.67\".toDouble() = $parsedDouble")
    
    // Safe parsing (ไม่ crash ถ้าแปลงไม่ได้)
    val safeInt = strInvalid.toIntOrNull()
    val safeDouble = strDouble.toDoubleOrNull()
    
    println("\nSafe Parsing:")
    println("\"abc\".toIntOrNull() = $safeInt")      // null
    println("\"45.67\".toDoubleOrNull() = $safeDouble") // 45.67
    
    // ============================================
    // Smart Cast
    // ============================================
    
    val obj: Any = "Hello, Kotlin!"
    
    if (obj is String) {
        // obj ถูก Smart Cast เป็น String อัตโนมัติใน block นี้
        println("\nSmart Cast:")
        println("Length: ${obj.length}")  // ไม่ต้อง cast เอง
        println("Upper: ${obj.uppercase()}")
    }
    
    // Smart Cast ใน when
    fun processValue(value: Any): String {
        return when (value) {
            is Int -> "เป็นตัวเลข Int: $value, สองเท่า = ${value * 2}"
            is String -> "เป็น String: $value, ความยาว = ${value.length}"
            is Boolean -> "เป็น Boolean: $value"
            is List<*> -> "เป็น List มี ${value.size} รายการ"
            else -> "ไม่รู้จักชนิดข้อมูล: ${value::class.simpleName}"
        }
    }
    
    println("\nSmart Cast with when:")
    println(processValue(42))
    println(processValue("Hello"))
    println(processValue(true))
    println(processValue(listOf(1, 2, 3)))
    println(processValue(3.14))
}
```

---

## ขั้นตอนที่ 16: Arithmetic Operators

```kotlin
fun main() {
    // ============================================
    // Arithmetic Operators (ตัวดำเนินการทางคณิตศาสตร์)
    // ============================================
    
    val a = 17
    val b = 5
    
    println("a = $a, b = $b")
    println("a + b = ${a + b}")   // 22 (บวก)
    println("a - b = ${a - b}")   // 12 (ลบ)
    println("a * b = ${a * b}")   // 85 (คูณ)
    println("a / b = ${a / b}")   // 3  (หาร - ผลลัพธ์เป็น Int จะตัดทศนิยม)
    println("a % b = ${a % b}")   // 2  (หารเอาเศษ/Modulo)
    
    // ระวัง! Int division ตัดทศนิยม
    println("\nInt Division:")
    println("17 / 5 = ${17 / 5}")      // 3 (ไม่ใช่ 3.4!)
    println("17.0 / 5 = ${17.0 / 5}")  // 3.4 (Double division)
    println("17 / 5.0 = ${17 / 5.0}")  // 3.4 (Double division)
    
    // ============================================
    // Assignment Operators
    // ============================================
    
    var x = 10
    println("\nAssignment Operators:")
    println("x = $x")
    
    x += 5    // x = x + 5 = 15
    println("x += 5  -> x = $x")
    
    x -= 3    // x = x - 3 = 12
    println("x -= 3  -> x = $x")
    
    x *= 2    // x = x * 2 = 24
    println("x *= 2  -> x = $x")
    
    x /= 4    // x = x / 4 = 6
    println("x /= 4  -> x = $x")
    
    x %= 4    // x = x % 4 = 2
    println("x %= 4  -> x = $x")
    
    // ============================================
    // Increment และ Decrement
    // ============================================
    
    var count = 5
    println("\nIncrement/Decrement:")
    println("count = $count")
    
    count++   // count = 6 (Post-increment)
    println("count++ -> $count")
    
    ++count   // count = 7 (Pre-increment)
    println("++count -> $count")
    
    count--   // count = 6 (Post-decrement)
    println("count-- -> $count")
    
    --count   // count = 5 (Pre-decrement)
    println("--count -> $count")
    
    // ============================================
    // Math Functions
    // ============================================
    
    import kotlin.math.*
    
    println("\nMath Functions:")
    println("abs(-5) = ${abs(-5)}")           // 5
    println("sqrt(16.0) = ${sqrt(16.0)}")     // 4.0
    println("pow(2.0, 10.0) = ${2.0.pow(10.0)}")  // 1024.0
    println("max(3, 7) = ${max(3, 7)}")       // 7
    println("min(3, 7) = ${min(3, 7)}")       // 3
    println("ceil(3.2) = ${ceil(3.2)}")       // 4.0
    println("floor(3.8) = ${floor(3.8)}")     // 3.0
    println("round(3.5) = ${round(3.5)}")     // 4.0
    println("PI = $PI")                        // 3.141592653589793
    println("E = $E")                          // 2.718281828459045
    println("log(100.0, 10.0) = ${log(100.0, 10.0)}")  // 2.0
    println("ln(E) = ${ln(E)}")               // 1.0
    println("sin(PI/2) = ${sin(PI/2)}")       // 1.0
    println("cos(0.0) = ${cos(0.0)}")         // 1.0
}
```

---

## ขั้นตอนที่ 17: Comparison และ Logical Operators

```kotlin
fun main() {
    // ============================================
    // Comparison Operators (ตัวดำเนินการเปรียบเทียบ)
    // ============================================
    
    val a = 10
    val b = 20
    
    println("a = $a, b = $b")
    println("a == b: ${a == b}")   // false (เท่ากัน)
    println("a != b: ${a != b}")   // true  (ไม่เท่ากัน)
    println("a < b: ${a < b}")     // true  (น้อยกว่า)
    println("a > b: ${a > b}")     // false (มากกว่า)
    println("a <= b: ${a <= b}")   // true  (น้อยกว่าหรือเท่ากัน)
    println("a >= b: ${a >= b}")   // false (มากกว่าหรือเท่ากัน)
    
    // ============================================
    // Structural vs Referential Equality
    // ============================================
    
    val str1 = "Hello"
    val str2 = "Hello"
    val str3 = str1
    
    println("\nString Equality:")
    println("str1 == str2: ${str1 == str2}")    // true (Structural - เนื้อหาเท่ากัน)
    println("str1 === str2: ${str1 === str2}")   // true (Referential - ชี้ที่เดียวกัน, String interned)
    println("str1 === str3: ${str1 === str3}")   // true
    
    // Object comparison
    data class Point(val x: Int, val y: Int)
    
    val p1 = Point(1, 2)
    val p2 = Point(1, 2)
    val p3 = p1
    
    println("\nObject Equality:")
    println("p1 == p2: ${p1 == p2}")    // true  (data class compares values)
    println("p1 === p2: ${p1 === p2}")  // false (different instances)
    println("p1 === p3: ${p1 === p3}")  // true  (same instance)
    
    // ============================================
    // Logical Operators
    // ============================================
    
    val isAdult = true
    val hasID = false
    val hasParentPermission = true
    
    println("\nLogical Operators:")
    println("isAdult && hasID: ${isAdult && hasID}")            // false
    println("isAdult || hasID: ${isAdult || hasID}")            // true
    println("!isAdult: ${!isAdult}")                             // false
    
    // Complex conditions
    val canEnter = isAdult || (hasParentPermission && !hasID)
    println("canEnter: $canEnter")  // true
    
    // ============================================
    // Bitwise Operators (สำหรับ Int และ Long)
    // ============================================
    
    val x = 0b1100  // 12 ในเลขฐาน 2
    val y = 0b1010  // 10 ในเลขฐาน 2
    
    println("\nBitwise Operators:")
    println("x = ${x.toString(2).padStart(4, '0')} (${x})")
    println("y = ${y.toString(2).padStart(4, '0')} (${y})")
    println("x and y = ${(x and y).toString(2).padStart(4, '0')} (${x and y})")  // 1000 = 8
    println("x or y  = ${(x or y).toString(2).padStart(4, '0')} (${x or y})")   // 1110 = 14
    println("x xor y = ${(x xor y).toString(2).padStart(4, '0')} (${x xor y})") // 0110 = 6
    println("x.inv() = ${x.inv()}")  // Bitwise NOT (two's complement)
    println("x shl 2 = ${x shl 2}")  // 48 (shift left 2 = multiply by 4)
    println("x shr 1 = ${x shr 1}")  // 6  (shift right 1 = divide by 2)
    println("x ushr 1 = ${x ushr 1}") // Unsigned shift right
}
```

---

## ขั้นตอนที่ 18: val vs var และ const

```kotlin
// ============================================
// val - Immutable reference (ค่าคงที่)
// ============================================

// val ไม่สามารถ reassign ได้
val PI = 3.14159
// PI = 3.14  // ❌ Error: Val cannot be reassigned

// แต่ val object สามารถเปลี่ยน state ข้างในได้
val mutableList = mutableListOf(1, 2, 3)
mutableList.add(4)    // ✅ ได้ (เปลี่ยน content ของ list)
// mutableList = mutableListOf(5, 6)  // ❌ ไม่ได้ (เปลี่ยน reference)

// ============================================
// var - Mutable reference (ตัวแปรที่เปลี่ยนได้)
// ============================================

var counter = 0
counter++         // ✅ ได้
counter = 10      // ✅ ได้
counter = -1      // ✅ ได้

// Best Practice: ใช้ val เสมอ ยกเว้นจำเป็นต้อง var
// ทำให้โค้ดปลอดภัยและอ่านง่ายขึ้น

// ============================================
// const val - Compile-time constant
// ============================================

// const val ต้องอยู่ใน top-level หรือ companion object
// และค่าต้องรู้ตอน compile time

const val MAX_RETRY = 3
const val APP_NAME = "MyApp"
const val VERSION = "1.0.0"

fun main() {
    println("App: $APP_NAME v$VERSION")
    println("Max retry: $MAX_RETRY")
    
    // const val ถูกแทนที่ตอน compile (inline)
    // ต่างจาก val ธรรมดาที่เป็น runtime constant
    
    // ============================================
    // lateinit var - Late initialization
    // ============================================
    
    // ใช้เมื่อต้องการ initialize ทีหลัง (ต้องเป็น var)
    lateinit var database: String
    
    // ตรวจสอบว่า initialize แล้วหรือยัง
    println("initialized: ${::database.isInitialized}")  // false
    
    database = "MyDatabase"
    println("initialized: ${::database.isInitialized}")  // true
    println("database: $database")
    
    // ============================================
    // Lazy initialization - by lazy
    // ============================================
    
    // คำนวณค่าเมื่อเรียกใช้ครั้งแรกเท่านั้น
    val expensiveValue: String by lazy {
        println("กำลังคำนวณ...")  // จะพิมพ์แค่ครั้งแรก
        "ผลลัพธ์ที่ใช้เวลานาน"
    }
    
    println("\nก่อนเข้าถึง lazy value")
    println(expensiveValue)  // พิมพ์ "กำลังคำนวณ..." แล้ว return ค่า
    println(expensiveValue)  // ใช้ค่าที่ cache ไว้ ไม่คำนวณอีก
}
```

---

## ขั้นตอนที่ 19: Numeric Literals และ Underscore

```kotlin
fun main() {
    // ============================================
    // Numeric Literals - การเขียนตัวเลขให้อ่านง่าย
    // ============================================
    
    // ใช้ _ คั่นตัวเลขให้อ่านง่าย
    val population = 70_000_000        // 70 ล้าน
    val distanceKm = 384_400           // ระยะทางถึงดวงจันทร์ (กม.)
    val creditCard = 1234_5678_9012_3456L
    
    println("ประชากรไทย: $population คน")
    println("ระยะทางถึงดวงจันทร์: $distanceKm กม.")
    
    // ============================================
    // Number Bases (ฐานตัวเลข)
    // ============================================
    
    // Decimal (ฐาน 10) - ปกติ
    val decimal = 255
    
    // Hexadecimal (ฐาน 16) - ขึ้นต้นด้วย 0x
    val hex = 0xFF    // = 255
    val hexColor = 0xFF5733  // RGB color
    
    // Binary (ฐาน 2) - ขึ้นต้นด้วย 0b
    val binary = 0b11111111  // = 255
    
    println("\nNumber Bases (ค่าเดียวกัน = 255):")
    println("Decimal: $decimal")
    println("Hex: $hex (0xFF)")
    println("Binary: $binary (0b11111111)")
    
    // แปลงระหว่างฐาน
    println("\nConversion:")
    println("255 in hex: ${255.toString(16).uppercase()}")  // FF
    println("255 in binary: ${255.toString(2)}")            // 11111111
    println("255 in octal: ${255.toString(8)}")             // 377
    
    // ============================================
    // Floating Point Precision
    // ============================================
    
    println("\nFloating Point Issues:")
    val a = 0.1
    val b = 0.2
    println("0.1 + 0.2 = ${a + b}")           // 0.30000000000000004 !!
    println("0.1 + 0.2 == 0.3: ${a + b == 0.3}") // false !!
    
    // วิธีแก้: ใช้ BigDecimal
    import java.math.BigDecimal
    val x = BigDecimal("0.1")
    val y = BigDecimal("0.2")
    println("\nBigDecimal:")
    println("0.1 + 0.2 = ${x + y}")           // 0.3
    println("0.1 + 0.2 == 0.3: ${x + y == BigDecimal("0.3")}") // true
    
    // หรือใช้ epsilon comparison
    fun Double.equalsDelta(other: Double, epsilon: Double = 1e-10): Boolean {
        return Math.abs(this - other) < epsilon
    }
    println("Using epsilon: ${(a + b).equalsDelta(0.3)}")  // true
}
```

---

## ขั้นตอนที่ 20: Ranges และ Operators เพิ่มเติม

```kotlin
fun main() {
    // ============================================
    // Range Operators
    // ============================================
    
    // IntRange
    val oneToTen = 1..10          // 1, 2, 3, ..., 10 (รวม 10)
    val oneToNine = 1 until 10    // 1, 2, 3, ..., 9 (ไม่รวม 10)
    val tenToOne = 10 downTo 1    // 10, 9, 8, ..., 1
    val evens = 0..20 step 2      // 0, 2, 4, ..., 20
    val odds = 1..20 step 2       // 1, 3, 5, ..., 19
    
    println("oneToTen: ${oneToTen.toList()}")
    println("oneToNine: ${oneToNine.toList()}")
    println("tenToOne: ${tenToOne.toList()}")
    println("evens: ${evens.toList()}")
    
    // CharRange
    val alphabet = 'a'..'z'
    println("\nAlphabet: ${alphabet.toList()}")
    
    val thaiVowels = 'า'..'ู'
    println("Thai vowels (some): ${thaiVowels.take(5).toList()}")
    
    // Check if in range
    val score = 85
    if (score in 80..89) {
        println("\nScore $score = Grade B")
    }
    
    val grade = when (score) {
        in 90..100 -> "A"
        in 80..89  -> "B"
        in 70..79  -> "C"
        in 60..69  -> "D"
        else       -> "F"
    }
    println("Grade: $grade")
    
    // ============================================
    // Elvis Operator (?:) - Preview
    // ============================================
    
    val nullableString: String? = null
    val nonNullString: String? = "Hello"
    
    val result1 = nullableString ?: "ค่าเริ่มต้น"
    val result2 = nonNullString ?: "ค่าเริ่มต้น"
    
    println("\nElvis Operator:")
    println("result1 = $result1")  // ค่าเริ่มต้น
    println("result2 = $result2")  // Hello
    
    // ============================================
    // Safe Call Operator (?.)
    // ============================================
    
    val nullStr: String? = null
    val validStr: String? = "Hello World"
    
    println("\nSafe Call:")
    println(nullStr?.length)    // null (ไม่ crash)
    println(validStr?.length)   // 11
    println(validStr?.uppercase()?.reversed())  // chaining
    
    // ============================================
    // Not-null Assertion (!!)
    // ============================================
    
    val notNull: String? = "ฉันแน่ใจว่าไม่ null"
    println("\nNot-null Assertion:")
    println(notNull!!.length)  // 23
    
    // ระวัง! ถ้า null จะ throw NullPointerException
    // val dangerous: String? = null
    // println(dangerous!!.length)  // 💥 KotlinNullPointerException!
    
    // ============================================
    // in และ !in Operator
    // ============================================
    
    val fruits = listOf("มะม่วง", "กล้วย", "ส้ม")
    
    println("\nin Operator:")
    println("มะม่วง in fruits: ${"มะม่วง" in fruits}")    // true
    println("แอปเปิ้ล in fruits: ${"แอปเปิ้ล" in fruits}") // false
    println("กล้วย !in fruits: ${"กล้วย" !in fruits}")    // false
    
    val num = 15
    println("15 in 10..20: ${num in 10..20}")  // true
    println("5 in 10..20: ${5 in 10..20}")     // false
}
```

---

## สรุป Part 02

```kotlin
// ตาราง Cheat Sheet - ชนิดข้อมูลใน Kotlin

/*
┌──────────────┬──────────┬─────────────────────────────┬──────────────────┐
│ ชนิดข้อมูล   │ ขนาด     │ ช่วงค่า                      │ ตัวอย่าง         │
├──────────────┼──────────┼─────────────────────────────┼──────────────────┤
│ Byte         │ 1 byte   │ -128 ถึง 127                 │ val b: Byte = 10 │
│ Short        │ 2 bytes  │ -32768 ถึง 32767             │ val s: Short = 10│
│ Int          │ 4 bytes  │ -2^31 ถึง 2^31-1             │ val i = 42       │
│ Long         │ 8 bytes  │ -2^63 ถึง 2^63-1             │ val l = 100L     │
│ Float        │ 4 bytes  │ ±3.4e38 (7 ตำแหน่ง)          │ val f = 3.14f    │
│ Double       │ 8 bytes  │ ±1.7e308 (15 ตำแหน่ง)        │ val d = 3.14     │
│ Boolean      │ -        │ true หรือ false               │ val b = true     │
│ Char         │ 2 bytes  │ Unicode character             │ val c = 'A'      │
│ String       │ varies   │ Unicode string                │ val s = "Hello"  │
└──────────────┴──────────┴─────────────────────────────┴──────────────────┘
*/
```

### แบบฝึกหัด Part 02

```kotlin
// แบบฝึกหัดที่ 1: ตรวจสอบว่าเป็นเลขคู่หรือคี่
fun isEven(n: Int): Boolean {
    // TODO: return true ถ้า n เป็นเลขคู่
    return TODO()
}

// แบบฝึกหัดที่ 2: แปลงอุณหภูมิ
fun celsiusToFahrenheit(celsius: Double): Double {
    // TODO: F = (C × 9/5) + 32
    return TODO()
}

fun fahrenheitToCelsius(fahrenheit: Double): Double {
    // TODO: C = (F - 32) × 5/9
    return TODO()
}

// แบบฝึกหัดที่ 3: คำนวณ compound interest
fun compoundInterest(principal: Double, rate: Double, years: Int): Double {
    // TODO: A = P × (1 + r)^n
    // principal = เงินต้น, rate = อัตราดอกเบี้ย (เช่น 0.05 = 5%), years = จำนวนปี
    return TODO()
}

fun main() {
    // ทดสอบ
    println("10 เป็นเลขคู่: ${isEven(10)}")    // true
    println("7 เป็นเลขคู่: ${isEven(7)}")       // false
    
    println("100°C = ${celsiusToFahrenheit(100.0)}°F")  // 212.0
    println("32°F = ${fahrenheitToCelsius(32.0)}°C")     // 0.0
    
    val finalAmount = compoundInterest(10000.0, 0.05, 10)
    println("เงินต้น 10,000 ดอกเบี้ย 5% ใน 10 ปี = %.2f".format(finalAmount))
    // ≈ 16288.95
}
```

### เฉลย

```kotlin
fun isEven(n: Int): Boolean = n % 2 == 0

fun celsiusToFahrenheit(celsius: Double): Double = (celsius * 9.0 / 5.0) + 32.0

fun fahrenheitToCelsius(fahrenheit: Double): Double = (fahrenheit - 32.0) * 5.0 / 9.0

fun compoundInterest(principal: Double, rate: Double, years: Int): Double {
    return principal * Math.pow(1 + rate, years.toDouble())
}
```

---

*Part 02 จบแล้ว | ก่อนหน้า: [Part 01](../part01/README.md) | ถัดไป: [Part 03](../part03/README.md)*
