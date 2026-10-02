# Part 10: Lambda and Higher-Order Functions

## ภาพรวม (Overview)
ในส่วนนี้เราจะเรียนรู้เกี่ยวกับ Lambda expressions และ Higher-Order Functions
ซึ่งเป็นหัวใจของ Functional Programming ใน Kotlin

**Steps ที่ครอบคลุม:** 201-230

---

## Step 201: Lambda Syntax พื้นฐาน

```kotlin
fun main() {
    // Lambda expression: code block ที่เป็น value
    // syntax: { parameters -> body }

    // 1. Lambda พื้นฐาน
    val hello = { println("Hello, Kotlin!") }
    hello()  // เรียกใช้

    // 2. Lambda ที่มี parameters
    val add = { a: Int, b: Int -> a + b }
    println("add(3, 4) = ${add(3, 4)}")

    val greet = { name: String -> "สวัสดี $name!" }
    println(greet("สมชาย"))

    // 3. Lambda ที่มี return type
    val square: (Int) -> Int = { x -> x * x }
    println("square(5) = ${square(5)}")

    // 4. Lambda หลายบรรทัด - ค่าสุดท้ายเป็น return value
    val complexCalc: (Int, Int) -> String = { a, b ->
        val sum = a + b
        val product = a * b
        val diff = a - b
        "sum=$sum, product=$product, diff=$diff"
    }
    println(complexCalc(4, 3))

    // 5. Lambda ที่ไม่มี parameters
    val getTime: () -> Long = { System.currentTimeMillis() }
    println("time: ${getTime()}")

    // 6. Lambda กับ Unit return type
    val printTwo: (String, Int) -> Unit = { str, num ->
        println("'$str' = $num")
    }
    printTwo("answer", 42)

    // 7. Lambda type syntax
    // (ParamType1, ParamType2) -> ReturnType
    val multiply: (Double, Double) -> Double = { x, y -> x * y }
    println("multiply: ${multiply(3.14, 2.0)}")

    // Lambda ใน variable
    val operations = mapOf(
        "add" to { a: Int, b: Int -> a + b },
        "sub" to { a: Int, b: Int -> a - b },
        "mul" to { a: Int, b: Int -> a * b }
    )

    println("\n--- Operations ---")
    val a = 10; val b = 3
    operations.forEach { (name, op) ->
        println("$name($a, $b) = ${op(a, b)}")
    }
}
```

---

## Step 202: Single Parameter Lambda - it

```kotlin
fun main() {
    // เมื่อ lambda มี parameter เดียว สามารถใช้ 'it' แทนชื่อได้

    // แบบปกติ
    val square1: (Int) -> Int = { x -> x * x }

    // ใช้ it
    val square2: (Int) -> Int = { it * it }

    println("square1(5) = ${square1(5)}")
    println("square2(5) = ${square2(5)}")

    // 'it' ใช้บ่อยมากใน standard library
    val numbers = listOf(1, 2, 3, 4, 5, 6, 7, 8, 9, 10)

    val doubled = numbers.map { it * 2 }
    val evens = numbers.filter { it % 2 == 0 }
    val sumSquares = numbers.map { it * it }.sum()

    println("\ndoubled: $doubled")
    println("evens: $evens")
    println("sumSquares: $sumSquares")

    // Chain operations
    val result = (1..20)
        .filter { it % 2 == 0 }       // กรองเลขคู่
        .map { it * it }               // ยกกำลัง 2
        .filter { it > 50 }            // กรองมากกว่า 50
        .take(5)                        // เอาแค่ 5 ตัว

    println("result: $result")

    // it กับ String operations
    val words = listOf("hello", "world", "kotlin", "lambda")
    
    val lengths = words.map { it.length }
    val longWords = words.filter { it.length > 5 }
    val uppercased = words.map { it.uppercase() }
    val startsWithK = words.filter { it.startsWith("k") }

    println("\nlengths: $lengths")
    println("longWords: $longWords")
    println("uppercased: $uppercased")
    println("startsWithK: $startsWithK")

    // it ใน when
    val values: List<Any> = listOf(1, "hello", 3.14, true, null)
    val descriptions = values.map {
        when (it) {
            is Int -> "Int: $it"
            is String -> "String: '$it'"
            is Double -> "Double: $it"
            is Boolean -> "Boolean: $it"
            null -> "null"
            else -> "unknown"
        }
    }
    descriptions.forEach { println(it) }
}
```

---

## Step 203: Higher-Order Functions

```kotlin
// Higher-Order Function = ฟังก์ชันที่รับ function เป็น parameter
// หรือคืน function เป็นผลลัพธ์

// รับ function เป็น parameter
fun applyTwice(n: Int, f: (Int) -> Int): Int {
    return f(f(n))
}

fun transform(list: List<Int>, transformer: (Int) -> Int): List<Int> {
    return list.map(transformer)
}

// รับ predicate function
fun <T> myFilter(list: List<T>, predicate: (T) -> Boolean): List<T> {
    val result = mutableListOf<T>()
    for (item in list) {
        if (predicate(item)) result.add(item)
    }
    return result
}

// รับหลาย functions
fun <T, R> myMap(list: List<T>, mapper: (T) -> R): List<R> {
    return list.map(mapper)
}

// คืน function เป็นผลลัพธ์
fun multiplier(factor: Int): (Int) -> Int {
    return { n -> n * factor }
}

fun adder(addend: Int): (Int) -> Int = { it + addend }

fun greeter(greeting: String): (String) -> String = { "$greeting, $it!" }

// Function ที่รับและคืน function (compose)
fun <A, B, C> compose(f: (B) -> C, g: (A) -> B): (A) -> C {
    return { x -> f(g(x)) }
}

fun main() {
    // applyTwice
    println("applyTwice(3, square) = ${applyTwice(3) { it * it }}")  // 81
    println("applyTwice(5, +3) = ${applyTwice(5) { it + 3 }}")        // 11

    // transform
    val numbers = listOf(1, 2, 3, 4, 5)
    println("\ntransform double: ${transform(numbers) { it * 2 }}")
    println("transform square: ${transform(numbers) { it * it }}")

    // myFilter
    println("\nmyFilter evens: ${myFilter(numbers) { it % 2 == 0 }}")
    println("myFilter >3: ${myFilter(numbers) { it > 3 }}")

    // Functions returning functions
    println("\n--- Returning Functions ---")
    val triple = multiplier(3)
    val quadruple = multiplier(4)
    println("triple(5) = ${triple(5)}")
    println("quadruple(5) = ${quadruple(5)}")

    val plus10 = adder(10)
    val plus100 = adder(100)
    println("plus10(5) = ${plus10(5)}")
    println("plus100(5) = ${plus100(5)}")

    val thaiGreet = greeter("สวัสดี")
    val engGreet = greeter("Hello")
    println(thaiGreet("สมชาย"))
    println(engGreet("World"))

    // Function composition
    println("\n--- Compose ---")
    val doubleIt = { x: Int -> x * 2 }
    val addThree = { x: Int -> x + 3 }
    
    val doubleAndAdd = compose(addThree, doubleIt)  // first double then add
    val addAndDouble = compose(doubleIt, addThree)  // first add then double
    
    println("doubleAndAdd(5) = ${doubleAndAdd(5)}")  // 5*2+3 = 13
    println("addAndDouble(5) = ${addAndDouble(5)}")  // (5+3)*2 = 16

    // สร้าง pipeline
    val pipeline = listOf<(Int) -> Int>(
        { it * 2 },
        { it + 10 },
        { it * it }
    ).reduce { f, g -> { x -> g(f(x)) } }
    
    println("pipeline(5) = ${pipeline(5)}")  // ((5*2)+10)^2 = 400
}
```

---

## Step 204: Closure - การ Capture ตัวแปร

```kotlin
fun main() {
    // Closure: lambda ที่ "capture" ตัวแปรจาก outer scope

    // 1. Capture val
    val multiplier = 3
    val multiplyBy3 = { n: Int -> n * multiplier }  // capture multiplier
    println("multiplyBy3(5) = ${multiplyBy3(5)}")
    println("multiplyBy3(7) = ${multiplyBy3(7)}")

    // 2. Capture var (mutable)
    var counter = 0
    val increment = { counter++ }
    val getCount = { counter }
    
    increment()
    increment()
    increment()
    println("\ncounter = ${getCount()}")  // 3

    // 3. Closure ใน factory function
    fun makeCounter(start: Int = 0, step: Int = 1): () -> Int {
        var current = start
        return { 
            val value = current
            current += step
            value
        }
    }
    
    val counter1 = makeCounter()
    val counter2 = makeCounter(10, 5)
    
    println("\ncounter1: ${counter1()}, ${counter1()}, ${counter1()}")
    println("counter2: ${counter2()}, ${counter2()}, ${counter2()}")

    // 4. Closure capture loop variable
    val actions = mutableListOf<() -> Unit>()
    
    // ผิด: capture index variable
    // for (i in 0..2) { actions.add { println(i) } }  // จะ print 3,3,3 ใน Java แต่ OK ใน Kotlin
    
    // Kotlin ถูกต้อง
    for (i in 0..2) {
        val index = i  // copy ค่า
        actions.add { println("action $index") }
    }
    actions.forEach { it() }

    // 5. Closure กับ memoization
    fun memoize(f: (Int) -> Int): (Int) -> Int {
        val cache = mutableMapOf<Int, Int>()
        return { n ->
            cache.getOrPut(n) {
                println("computing f($n)...")
                f(n)
            }
        }
    }
    
    val expensiveFn = { n: Int -> 
        Thread.sleep(100)  // จำลองการคำนวณที่ใช้เวลา
        n * n + n
    }
    
    val memoized = memoize(expensiveFn)
    
    println("\n--- Memoization ---")
    println("f(5) = ${memoized(5)}")  // คำนวณ
    println("f(5) = ${memoized(5)}")  // ใช้ cache
    println("f(3) = ${memoized(3)}")  // คำนวณ
    println("f(5) = ${memoized(5)}")  // ใช้ cache
    println("f(3) = ${memoized(3)}")  // ใช้ cache
}
```

---

## Step 205: Function References

```kotlin
fun double(n: Int) = n * 2
fun isEven(n: Int) = n % 2 == 0
fun toUpperCase(s: String) = s.uppercase()
fun printItem(item: Any) = println("Item: $item")

class Calculator {
    fun add(a: Int, b: Int) = a + b
    fun multiply(a: Int, b: Int) = a * b
    
    companion object {
        fun square(n: Int) = n * n
    }
}

data class Person(val name: String, val age: Int)

fun main() {
    // Function Reference: ใช้ :: เพื่ออ้างอิง function

    // 1. Top-level function reference
    val numbers = listOf(1, 2, 3, 4, 5)
    
    val doubled = numbers.map(::double)         // แทน { it * 2 }
    val evens = numbers.filter(::isEven)         // แทน { it % 2 == 0 }
    
    println("doubled: $doubled")
    println("evens: $evens")

    // 2. Member function reference
    val words = listOf("hello", "world", "kotlin")
    val uppercased = words.map(String::uppercase)  // String.uppercase()
    val lengths = words.map(String::length)         // String.length property
    
    println("\nuppercased: $uppercased")
    println("lengths: $lengths")

    // 3. Instance function reference
    val calc = Calculator()
    val addFn: (Int, Int) -> Int = calc::add
    val mulFn: (Int, Int) -> Int = calc::multiply
    
    println("\nadd(3, 4) = ${addFn(3, 4)}")
    println("multiply(3, 4) = ${mulFn(3, 4)}")

    // 4. Companion object reference
    val squareFn: (Int) -> Int = Calculator::square
    println("square(5) = ${squareFn(5)}")

    // 5. Constructor reference
    data class Point(val x: Double, val y: Double)
    
    val pointCreator: (Double, Double) -> Point = ::Point
    val point = pointCreator(3.0, 4.0)
    println("\npoint: $point")
    
    val points = listOf(1.0, 2.0, 3.0).map { x -> ::Point.call(x, x * 2) }
    println("points: $points")

    // 6. Property reference
    val people = listOf(
        Person("สมชาย", 25),
        Person("สมหญิง", 30),
        Person("สมศักดิ์", 22)
    )
    
    val names = people.map(Person::name)
    val ages = people.map(Person::age)
    
    println("\nnames: $names")
    println("ages: $ages")
    
    val sorted = people.sortedBy(Person::age)
    println("sorted by age: ${sorted.map { it.name }}")

    // 7. forEach กับ ::
    numbers.forEach(::println)
    println()
    words.forEach(::printItem)
}
```

---

## Step 206: Trailing Lambda และ Scope Functions

```kotlin
// Trailing lambda: ถ้า parameter สุดท้ายเป็น function
// สามารถวางไว้นอก () ได้

fun withLogging(name: String, block: () -> Unit) {
    println(">> เริ่ม $name")
    block()
    println("<< จบ $name")
}

fun <T> measure(name: String, block: () -> T): Pair<T, Long> {
    val start = System.currentTimeMillis()
    val result = block()
    val elapsed = System.currentTimeMillis() - start
    return Pair(result, elapsed)
}

fun retry(maxAttempts: Int, action: (Int) -> Boolean): Boolean {
    for (attempt in 1..maxAttempts) {
        println("พยายามครั้งที่ $attempt")
        if (action(attempt)) return true
    }
    return false
}

fun main() {
    // Trailing lambda
    withLogging("Task A") {
        println("  ทำงาน A...")
        Thread.sleep(50)
    }
    
    withLogging("Task B") {
        println("  ทำงาน B...")
    }

    // measure
    println()
    val (result, time) = measure("sorting") {
        (1..1000).toList().shuffled().sorted()
    }
    println("Sorted ${result.size} items in ${time}ms")

    // retry
    println()
    var count = 0
    val success = retry(5) { attempt ->
        count++
        attempt >= 3  // สำเร็จในครั้งที่ 3
    }
    println("success: $success, tried: $count times")

    // Scope functions ทบทวน
    println("\n--- Scope Functions ---")

    // let - transform, null check
    val name: String? = "  สมชาย  "
    val cleaned = name?.let { it.trim() }?.let { it.lowercase() }
    println("cleaned: $cleaned")

    // run - execute block, return last expression
    val greeting = run {
        val prefix = "สวัสดี"
        val suffix = "Kotlin"
        "$prefix, $suffix!"
    }
    println("greeting: $greeting")

    // apply - configure object, return same object
    data class Config(var host: String = "", var port: Int = 0, var debug: Boolean = false)
    
    val config = Config().apply {
        host = "localhost"
        port = 8080
        debug = true
    }
    println("config: $config")

    // also - side effects, return same object  
    val numbers = mutableListOf(3, 1, 4, 1, 5)
        .also { println("original: $it") }
        .apply { sort() }
        .also { println("sorted: $it") }
    
    // with - like apply but not extension
    val message = with(config) {
        "Connecting to $host:$port (debug=$debug)"
    }
    println("message: $message")
}
```

---

## Step 207: inline Functions

```kotlin
// inline function: compiler แทรก code ของ function ลงไปตรงๆ
// ไม่มี overhead ของ function call
// เหมาะกับ Higher-Order Functions ที่เรียกบ่อย

inline fun <T> measureTime(block: () -> T): T {
    val start = System.currentTimeMillis()
    val result = block()
    val end = System.currentTimeMillis()
    println("ใช้เวลา: ${end - start}ms")
    return result
}

inline fun <T> logExecution(name: String, block: () -> T): T {
    println("กำลัง execute: $name")
    val result = block()
    println("เสร็จสิ้น: $name")
    return result
}

// reified type parameter กับ inline
inline fun <reified T> printType(value: Any) {
    if (value is T) {
        println("$value เป็น ${T::class.simpleName}")
    } else {
        println("$value ไม่ใช่ ${T::class.simpleName}")
    }
}

inline fun <reified T> List<Any>.filterType(): List<T> {
    return filterIsInstance<T>()
}

// inline property
inline val currentTimeMs: Long get() = System.currentTimeMillis()

fun main() {
    // measureTime
    val result = measureTime {
        (1..1000000).sum()
    }
    println("result: $result")

    println()

    // logExecution
    val data = logExecution("Fetch Users") {
        listOf("สมชาย", "สมหญิง", "สมศักดิ์")
    }
    println("data: $data")

    println()

    // reified
    val mixed: List<Any> = listOf(1, "hello", 3.14, "world", true, 2)
    val strings = mixed.filterType<String>()
    val ints = mixed.filterType<Int>()
    println("strings: $strings")
    println("ints: $ints")

    println()

    printType<String>("hello")
    printType<Int>("hello")
    printType<Int>(42)

    // inline ช่วยให้ใช้ non-local return ได้
    println("\n--- Non-local return ---")
    
    fun findFirst(list: List<Int>, predicate: (Int) -> Boolean): Int? {
        list.forEach { item ->
            if (predicate(item)) return item  // return ออกจาก findFirst (เพราะ forEach เป็น inline)
        }
        return null
    }
    
    val firstEven = findFirst(listOf(1, 3, 4, 6, 7)) { it % 2 == 0 }
    println("firstEven: $firstEven")

    // inline property
    println("currentTime: $currentTimeMs")
}
```

---

## Step 208: noinline และ crossinline

```kotlin
// noinline: ป้องกัน inline สำหรับ lambda parameter ที่ต้องการ
// ใช้เมื่อต้องการเก็บ lambda เป็น object

inline fun runWithCallbacks(
    action: () -> Unit,
    noinline onSuccess: () -> Unit,  // noinline: เพราะต้องส่งต่อให้ Thread
    noinline onError: (Exception) -> Unit
) {
    try {
        action()
        // ส่ง onSuccess ต่อให้ Thread (ต้องเป็น object)
        Thread { onSuccess() }.start()
    } catch (e: Exception) {
        Thread { onError(e) }.start()
    }
}

// crossinline: lambda ห้ามทำ non-local return
// ใช้เมื่อ lambda จะถูก execute ใน context อื่น

inline fun runAsync(crossinline block: () -> Unit) {
    Thread {
        block()  // crossinline ป้องกัน return ออกจาก outer function
    }.start()
}

// ตัวอย่างจริง: Higher-Order Function ที่ซับซ้อน
inline fun <T, R> List<T>.transformEach(
    transform: (T) -> R,
    noinline onComplete: (List<R>) -> Unit
): List<R> {
    val result = map(transform)
    onComplete(result)
    return result
}

fun main() {
    println("--- noinline ---")
    runWithCallbacks(
        action = { println("กำลังทำงาน...") },
        onSuccess = { println("สำเร็จ!") },
        onError = { e -> println("Error: ${e.message}") }
    )
    
    Thread.sleep(100)  // รอ thread

    println("\n--- crossinline ---")
    runAsync { println("Async execution") }
    
    Thread.sleep(100)

    println("\n--- transformEach ---")
    val numbers = listOf(1, 2, 3, 4, 5)
    val results = numbers.transformEach(
        transform = { it * it },
        onComplete = { list -> println("Complete! results: $list") }
    )
    println("returned: $results")

    // เปรียบเทียบ: inline vs non-inline performance
    println("\n--- Performance Comparison ---")
    
    // Non-inline: สร้าง lambda object ทุกครั้ง
    fun <T, R> nonInlineMap(list: List<T>, f: (T) -> R): List<R> = list.map(f)
    
    // Inline: ไม่มี overhead
    inline fun <T, R> inlineMap(list: List<T>, f: (T) -> R): List<R> = list.map(f)
    
    val bigList = (1..100000).toList()
    
    val t1 = System.currentTimeMillis()
    repeat(1000) { nonInlineMap(bigList) { it * 2 } }
    val t1e = System.currentTimeMillis() - t1
    
    val t2 = System.currentTimeMillis()
    repeat(1000) { inlineMap(bigList) { it * 2 } }
    val t2e = System.currentTimeMillis() - t2
    
    println("non-inline: ${t1e}ms")
    println("inline: ${t2e}ms")
}
```

---

## Step 209: Currying และ Partial Application

```kotlin
// Currying: แปลง f(a, b, c) เป็น f(a)(b)(c)

fun <A, B, C> curry(f: (A, B) -> C): (A) -> (B) -> C {
    return { a -> { b -> f(a, b) } }
}

fun <A, B, C, D> curry3(f: (A, B, C) -> D): (A) -> (B) -> (C) -> D {
    return { a -> { b -> { c -> f(a, b, c) } } }
}

// Partial Application: กำหนดบาง arguments ล่วงหน้า
fun <A, B, C> partial(f: (A, B) -> C, a: A): (B) -> C {
    return { b -> f(a, b) }
}

fun main() {
    // Currying
    val add = { a: Int, b: Int -> a + b }
    val curriedAdd = curry(add)
    
    val add5 = curriedAdd(5)  // partial application
    println("add5(3) = ${add5(3)}")
    println("add5(10) = ${add5(10)}")
    
    val curriedAdd3 = curry3 { a: Int, b: Int, c: Int -> a + b + c }
    println("\ncurriedAdd3(1)(2)(3) = ${curriedAdd3(1)(2)(3)}")
    
    val addTo10 = curriedAdd3(10)
    val addTo10And5 = addTo10(5)
    println("addTo10And5(7) = ${addTo10And5(7)}")

    // Partial Application
    fun multiply(a: Int, b: Int) = a * b
    
    val double = partial(::multiply, 2)
    val triple = partial(::multiply, 3)
    val times10 = partial(::multiply, 10)
    
    println("\ndouble(5) = ${double(5)}")
    println("triple(5) = ${triple(5)}")
    println("times10(5) = ${times10(5)}")
    
    val numbers = (1..5).toList()
    println("doubled: ${numbers.map(double)}")
    println("tripled: ${numbers.map(triple)}")

    // Practical example: Builder with functions
    data class HttpRequest(val method: String, val url: String, val body: String?)
    
    val get = partial({ method: String, url: String -> HttpRequest(method, url, null) }, "GET")
    val post: (String, String?) -> HttpRequest = { url, body -> HttpRequest("POST", url, body) }
    
    val getUsers = partial(get, "/api/users")
    val postUser = { body: String? -> post("/api/users", body) }
    
    println("\n${getUsers()}")
    println("${postUser("""{"name":"สมชาย"}""")}")
}
```

---

## Step 210: Lambda กับ Collections ขั้นสูง

```kotlin
data class Transaction(
    val id: Int,
    val type: String,  // "deposit", "withdraw", "transfer"
    val amount: Double,
    val fromAccount: String,
    val toAccount: String?,
    val timestamp: Long
)

fun main() {
    val transactions = listOf(
        Transaction(1, "deposit", 5000.0, "ACC001", null, 1000L),
        Transaction(2, "withdraw", 1500.0, "ACC001", null, 2000L),
        Transaction(3, "transfer", 2000.0, "ACC001", "ACC002", 3000L),
        Transaction(4, "deposit", 10000.0, "ACC002", null, 4000L),
        Transaction(5, "transfer", 3000.0, "ACC002", "ACC003", 5000L),
        Transaction(6, "withdraw", 500.0, "ACC001", null, 6000L),
        Transaction(7, "deposit", 8000.0, "ACC003", null, 7000L),
        Transaction(8, "transfer", 1000.0, "ACC003", "ACC001", 8000L)
    )

    // Complex analysis
    println("=== Transaction Analysis ===")

    // รายได้/รายจ่ายตาม account
    val accountBalance = transactions.groupBy { it.fromAccount }
        .mapValues { (_, txns) ->
            txns.sumOf { when (it.type) {
                "deposit" -> it.amount
                "withdraw" -> -it.amount
                "transfer" -> -it.amount
                else -> 0.0
            }}
        }
    
    // เพิ่มยอดรับ transfer
    val transfersReceived = transactions
        .filter { it.type == "transfer" && it.toAccount != null }
        .groupBy { it.toAccount!! }
        .mapValues { (_, txns) -> txns.sumOf { it.amount } }

    println("ยอดเงิน (ไม่รวม transfer received):")
    accountBalance.forEach { (acc, balance) ->
        val received = transfersReceived[acc] ?: 0.0
        println("  $acc: ${balance + received}")
    }

    // Top transactions ตาม amount
    val top3 = transactions.sortedByDescending { it.amount }.take(3)
    println("\nTop 3 transactions:")
    top3.forEach { println("  ${it.id}: ${it.type} ${it.amount} (${it.fromAccount})") }

    // ใช้ fold สร้าง running balance
    println("\nRunning balance ACC001:")
    transactions
        .filter { it.fromAccount == "ACC001" || it.toAccount == "ACC001" }
        .sortedBy { it.timestamp }
        .fold(0.0) { balance, txn ->
            val change = when {
                txn.fromAccount == "ACC001" && txn.type == "deposit" -> txn.amount
                txn.fromAccount == "ACC001" -> -txn.amount
                txn.toAccount == "ACC001" -> txn.amount
                else -> 0.0
            }
            val newBalance = balance + change
            println("  txn ${txn.id}: ${if (change >= 0) "+" else ""}$change -> $newBalance")
            newBalance
        }

    // Lambda เป็น strategy
    println("\n--- Sorting Strategies ---")
    
    val strategies: Map<String, Comparator<Transaction>> = mapOf(
        "amount" to compareByDescending { it.amount },
        "time" to compareBy { it.timestamp },
        "type" to compareBy { it.type }
    )
    
    strategies.forEach { (name, comparator) ->
        val sorted = transactions.sortedWith(comparator)
        println("By $name: ${sorted.map { it.id }}")
    }
}
```

---

## Step 211: Functional Programming Patterns

```kotlin
// Pure functions - ไม่มี side effects
fun add(a: Int, b: Int): Int = a + b
fun doubleAll(list: List<Int>): List<Int> = list.map { it * 2 }

// Immutable data transformation
data class AppState(
    val users: List<String> = emptyList(),
    val loading: Boolean = false,
    val error: String? = null
)

sealed class Action {
    object StartLoading : Action()
    data class LoadSuccess(val users: List<String>) : Action()
    data class LoadError(val message: String) : Action()
    data class AddUser(val name: String) : Action()
    data class RemoveUser(val name: String) : Action()
}

// Pure reducer function
fun reduce(state: AppState, action: Action): AppState {
    return when (action) {
        is Action.StartLoading -> state.copy(loading = true, error = null)
        is Action.LoadSuccess -> state.copy(loading = false, users = action.users)
        is Action.LoadError -> state.copy(loading = false, error = action.message)
        is Action.AddUser -> state.copy(users = state.users + action.name)
        is Action.RemoveUser -> state.copy(users = state.users - action.name)
    }
}

// Store ที่ใช้ reducer
class Store(initialState: AppState) {
    private var state = initialState
    private val listeners = mutableListOf<(AppState) -> Unit>()
    
    fun getState() = state
    
    fun dispatch(action: Action) {
        state = reduce(state, action)
        listeners.forEach { it(state) }
    }
    
    fun subscribe(listener: (AppState) -> Unit) {
        listeners.add(listener)
    }
}

fun main() {
    // Redux-like state management
    println("=== Redux-like State ===")
    
    val store = Store(AppState())
    
    store.subscribe { state ->
        println("State update:")
        println("  loading=${state.loading}, error=${state.error}")
        println("  users=${state.users}")
    }
    
    store.dispatch(Action.StartLoading)
    store.dispatch(Action.LoadSuccess(listOf("สมชาย", "สมหญิง", "สมศักดิ์")))
    store.dispatch(Action.AddUser("สมใจ"))
    store.dispatch(Action.RemoveUser("สมหญิง"))
    store.dispatch(Action.LoadError("Connection failed"))

    // Functional pipeline
    println("\n=== Functional Pipeline ===")
    
    data class Student(val name: String, val grade: Int, val passed: Boolean)
    
    val students = listOf(
        Student("อ", 85, true),
        Student("ข", 45, false),
        Student("ค", 92, true),
        Student("ง", 55, false),
        Student("จ", 78, true),
        Student("ฉ", 38, false)
    )
    
    // Functional transformation pipeline
    val analysis = students
        .partition { it.passed }
        .let { (passed, failed) ->
            mapOf(
                "passed" to passed,
                "failed" to failed,
                "passRate" to listOf(passed.size * 100.0 / students.size),
                "avgGrade" to listOf(students.map { it.grade }.average())
            )
        }
    
    println("ผ่าน: ${(analysis["passed"] as List<*>).size} คน")
    println("ไม่ผ่าน: ${(analysis["failed"] as List<*>).size} คน")
    println("Pass rate: ${(analysis["passRate"] as List<*>)[0]}%")
    println("เฉลี่ย: ${(analysis["avgGrade"] as List<*>)[0]}")
}
```

---

## Step 212: Lambda กับ Coroutines Preview

```kotlin
// Preview: Lambda ใน async code

typealias Callback<T> = (Result<T>) -> Unit

class AsyncRunner {
    // ฟังก์ชันที่รับ lambda เป็น callback
    fun <T> execute(
        operation: () -> T,
        onSuccess: (T) -> Unit,
        onError: (Exception) -> Unit = { e -> println("Error: ${e.message}") }
    ) {
        try {
            val result = operation()
            onSuccess(result)
        } catch (e: Exception) {
            onError(e)
        }
    }
    
    fun <T> executeWithCallback(operation: () -> T, callback: Callback<T>) {
        try {
            callback(Result.success(operation()))
        } catch (e: Exception) {
            callback(Result.failure(e))
        }
    }
}

// Builder pattern ด้วย lambda
class RequestBuilder {
    private var url = ""
    private var method = "GET"
    private var body: String? = null
    private val headers = mutableMapOf<String, String>()
    private var timeout = 30
    private var retries = 0
    private var onSuccess: ((String) -> Unit)? = null
    private var onError: ((Exception) -> Unit)? = null
    private var onFinally: (() -> Unit)? = null
    
    fun url(url: String) = apply { this.url = url }
    fun method(method: String) = apply { this.method = method }
    fun body(body: String) = apply { this.body = body }
    fun header(key: String, value: String) = apply { headers[key] = value }
    fun timeout(seconds: Int) = apply { timeout = seconds }
    fun retry(times: Int) = apply { retries = times }
    fun onSuccess(handler: (String) -> Unit) = apply { onSuccess = handler }
    fun onError(handler: (Exception) -> Unit) = apply { onError = handler }
    fun onFinally(handler: () -> Unit) = apply { onFinally = handler }
    
    fun execute() {
        println("Executing $method $url")
        headers.forEach { (k, v) -> println("  $k: $v") }
        body?.let { println("  body: $it") }
        
        // จำลองการ execute
        try {
            val response = "Response from $url"
            onSuccess?.invoke(response)
        } catch (e: Exception) {
            onError?.invoke(e)
        } finally {
            onFinally?.invoke()
        }
    }
}

fun main() {
    val runner = AsyncRunner()
    
    println("=== Async Runner ===")
    
    runner.execute(
        operation = { (1..100).sum() },
        onSuccess = { result -> println("Sum: $result") }
    )
    
    runner.execute(
        operation = { throw RuntimeException("Something went wrong") },
        onSuccess = { println("Won't reach here") },
        onError = { e -> println("Caught: ${e.message}") }
    )
    
    runner.executeWithCallback({ "Hello, World!" }) { result ->
        result.fold(
            onSuccess = { println("Success: $it") },
            onFailure = { println("Error: ${it.message}") }
        )
    }
    
    println("\n=== Request Builder ===")
    
    RequestBuilder()
        .url("https://api.example.com/users")
        .method("POST")
        .header("Authorization", "Bearer token-123")
        .header("Content-Type", "application/json")
        .body("""{"name": "สมชาย"}""")
        .timeout(60)
        .retry(3)
        .onSuccess { response -> println("Got: $response") }
        .onError { e -> println("Failed: ${e.message}") }
        .onFinally { println("Request completed") }
        .execute()
}
```

---

## Step 213: Lambda Type Aliases

```kotlin
// typealias ทำให้ code อ่านง่ายขึ้น

typealias StringTransformer = (String) -> String
typealias IntPredicate = (Int) -> Boolean
typealias EventHandler = (String, Map<String, Any>) -> Unit
typealias AsyncCallback<T> = (Result<T>) -> Unit
typealias Reducer<S, A> = (S, A) -> S

// ใช้ typealias
fun applyTransformations(text: String, transformers: List<StringTransformer>): String {
    return transformers.fold(text) { acc, transform -> transform(acc) }
}

fun processNumbers(numbers: List<Int>, predicate: IntPredicate): List<Int> {
    return numbers.filter(predicate)
}

class EventSystem {
    private val handlers = mutableMapOf<String, MutableList<EventHandler>>()
    
    fun on(event: String, handler: EventHandler) {
        handlers.getOrPut(event) { mutableListOf() }.add(handler)
    }
    
    fun emit(event: String, data: Map<String, Any> = emptyMap()) {
        handlers[event]?.forEach { it(event, data) }
    }
}

fun main() {
    // applyTransformations
    val transformers: List<StringTransformer> = listOf(
        { it.trim() },
        { it.lowercase() },
        { it.replace(" ", "_") },
        { if (it.length > 20) it.substring(0, 20) else it }
    )
    
    val inputs = listOf(
        "  Hello World  ",
        "  KOTLIN IS AWESOME  ",
        "  THIS IS A VERY LONG STRING THAT NEEDS TRUNCATING  "
    )
    
    println("=== String Transformations ===")
    inputs.forEach { input ->
        val result = applyTransformations(input, transformers)
        println("'$input' -> '$result'")
    }

    // processNumbers
    val numbers = (1..20).toList()
    val predicates: Map<String, IntPredicate> = mapOf(
        "even" to { it % 2 == 0 },
        "prime" to { n -> n > 1 && (2..Math.sqrt(n.toDouble()).toInt()).none { n % it == 0 } },
        "fibonacci" to { n ->
            fun isFib(n: Int): Boolean {
                fun isPerfectSquare(x: Int) = Math.sqrt(x.toDouble()).toInt().let { it * it == x }
                return isPerfectSquare(5 * n * n + 4) || isPerfectSquare(5 * n * n - 4)
            }
            isFib(n)
        }
    )
    
    println("\n=== Number Processing ===")
    predicates.forEach { (name, pred) ->
        println("$name: ${processNumbers(numbers, pred)}")
    }

    // EventSystem
    println("\n=== Event System ===")
    val events = EventSystem()
    
    events.on("click") { event, data ->
        println("Click handler: x=${data["x"]}, y=${data["y"]}")
    }
    
    events.on("click") { event, data ->
        println("Click logger: event=$event")
    }
    
    events.on("submit") { event, data ->
        println("Submit: form=${data["form"]}")
    }
    
    events.emit("click", mapOf("x" to 100, "y" to 200))
    events.emit("submit", mapOf("form" to "login"))
    events.emit("hover")  // no handler
}
```

---

## Step 214: แบบฝึกหัด Lambda ขั้นสูง

```kotlin
// ============================================================
// Exercise 1: Functional Calculator
// ============================================================

typealias Operation = (Double, Double) -> Double
typealias UnaryOperation = (Double) -> Double

class FunctionalCalculator {
    private val history = mutableListOf<String>()
    
    private val operations: Map<String, Operation> = mapOf(
        "+" to { a, b -> a + b },
        "-" to { a, b -> a - b },
        "*" to { a, b -> a * b },
        "/" to { a, b -> if (b != 0.0) a / b else throw ArithmeticException("Division by zero") },
        "^" to { a, b -> Math.pow(a, b) },
        "%" to { a, b -> a % b }
    )
    
    private val unaryOperations: Map<String, UnaryOperation> = mapOf(
        "sqrt" to { Math.sqrt(it) },
        "abs" to { Math.abs(it) },
        "neg" to { -it },
        "floor" to { Math.floor(it) },
        "ceil" to { Math.ceil(it) }
    )
    
    fun calculate(a: Double, op: String, b: Double): Double {
        val operation = operations[op] ?: throw IllegalArgumentException("Unknown operation: $op")
        val result = operation(a, b)
        history.add("$a $op $b = $result")
        return result
    }
    
    fun apply(a: Double, op: String): Double {
        val operation = unaryOperations[op] ?: throw IllegalArgumentException("Unknown operation: $op")
        val result = operation(a)
        history.add("$op($a) = $result")
        return result
    }
    
    fun getHistory() = history.toList()
    
    // สร้าง pipeline ของ operations
    fun pipeline(initial: Double, vararg ops: Pair<String, Double>): Double {
        return ops.fold(initial) { acc, (op, operand) -> calculate(acc, op, operand) }
    }
}

// ============================================================
// Exercise 2: Event-Driven Architecture
// ============================================================

data class Event(val type: String, val payload: Any?, val timestamp: Long = System.currentTimeMillis())

typealias Middleware = (Event, (Event) -> Unit) -> Unit

class EventPipeline {
    private val middlewares = mutableListOf<Middleware>()
    private val handlers = mutableMapOf<String, MutableList<(Event) -> Unit>>()
    
    fun use(middleware: Middleware) = apply { middlewares.add(middleware) }
    
    fun on(type: String, handler: (Event) -> Unit) = apply {
        handlers.getOrPut(type) { mutableListOf() }.add(handler)
    }
    
    fun dispatch(event: Event) {
        val chain = middlewares.foldRight(
            { e: Event ->
                handlers[e.type]?.forEach { it(e) }
            }
        ) { middleware, next ->
            { e: Event -> middleware(e, next) }
        }
        chain(event)
    }
}

fun main() {
    // Functional Calculator
    println("=== Functional Calculator ===")
    val calc = FunctionalCalculator()
    
    println("10 + 5 = ${calc.calculate(10.0, "+", 5.0)}")
    println("10 - 3 = ${calc.calculate(10.0, "-", 3.0)}")
    println("6 * 7 = ${calc.calculate(6.0, "*", 7.0)}")
    println("15 / 4 = ${calc.calculate(15.0, "/", 4.0)}")
    println("2 ^ 8 = ${calc.calculate(2.0, "^", 8.0)}")
    println("sqrt(144) = ${calc.apply(144.0, "sqrt")}")
    println("abs(-42) = ${calc.apply(-42.0, "abs")}")
    
    // Pipeline
    val result = calc.pipeline(100.0, "-" to 20.0, "*" to 1.5, "+" to 50.0)
    println("pipeline: $result")
    
    println("\nHistory:")
    calc.getHistory().forEach { println("  $it") }
    
    // Event Pipeline
    println("\n=== Event Pipeline ===")
    
    // Logging middleware
    val loggingMiddleware: Middleware = { event, next ->
        println("[LOG] Dispatching: ${event.type}")
        next(event)
        println("[LOG] Done: ${event.type}")
    }
    
    // Validation middleware
    val validationMiddleware: Middleware = { event, next ->
        when {
            event.type.isBlank() -> println("[VALIDATE] ปฏิเสธ: event type ว่าง")
            event.type == "blocked" -> println("[VALIDATE] ปฏิเสธ: blocked event")
            else -> next(event)
        }
    }
    
    // Timing middleware
    val timingMiddleware: Middleware = { event, next ->
        val start = System.currentTimeMillis()
        next(event)
        println("[TIMING] ${event.type}: ${System.currentTimeMillis() - start}ms")
    }
    
    val pipeline = EventPipeline()
        .use(loggingMiddleware)
        .use(validationMiddleware)
        .use(timingMiddleware)
        .on("user.created") { e -> println("  Handler: New user: ${e.payload}") }
        .on("order.placed") { e -> println("  Handler: New order: ${e.payload}") }
        .on("user.created") { e -> println("  Handler: Send welcome email to ${e.payload}") }
    
    pipeline.dispatch(Event("user.created", "สมชาย"))
    println()
    pipeline.dispatch(Event("order.placed", mapOf("item" to "iPhone", "qty" to 2)))
    println()
    pipeline.dispatch(Event("blocked", null))
    println()
    pipeline.dispatch(Event("", null))
}
```

---

## Step 215: สรุป Lambda และ Higher-Order Functions

```kotlin
// ============================================================
// Summary: All Lambda Patterns
// ============================================================

fun main() {
    println("=== Lambda Syntax Summary ===\n")

    // 1. Basic lambda
    val lambda1: () -> Unit = { println("1. Basic lambda") }
    lambda1()

    // 2. Lambda with parameters
    val lambda2: (Int, Int) -> Int = { a, b -> a + b }
    println("2. lambda2(3, 4) = ${lambda2(3, 4)}")

    // 3. Single param with 'it'
    val lambda3: (String) -> Int = { it.length }
    println("3. lambda3('hello') = ${lambda3("hello")}")

    // 4. Multi-line lambda
    val lambda4: (List<Int>) -> Map<String, Any> = { list ->
        val sum = list.sum()
        val avg = list.average()
        mapOf("sum" to sum, "avg" to avg, "count" to list.size)
    }
    println("4. ${lambda4(listOf(1,2,3,4,5))}")

    // 5. Function reference
    println("5. map with ::println:")
    listOf(1, 2, 3).forEach(::println)

    // 6. Higher-order function
    fun applyToList(list: List<Int>, f: (Int) -> Int) = list.map(f)
    println("6. applyToList: ${applyToList(listOf(1,2,3)) { it * 10 }}")

    // 7. Returning function
    fun multiplierOf(n: Int): (Int) -> Int = { it * n }
    val triple = multiplierOf(3)
    println("7. triple(7) = ${triple(7)}")

    // 8. Closure
    var sum = 0
    val addToSum = { n: Int -> sum += n }
    (1..5).forEach(addToSum)
    println("8. sum = $sum")

    // 9. Inline function
    inline fun <T> timed(block: () -> T): T {
        val s = System.currentTimeMillis()
        val r = block()
        println("9. Time: ${System.currentTimeMillis() - s}ms")
        return r
    }
    timed { (1..1000000).sum() }

    // 10. SAM / fun interface
    fun interface Predicate { fun test(n: Int): Boolean }
    val isEven = Predicate { it % 2 == 0 }
    println("10. isEven(4) = ${isEven.test(4)}, isEven(3) = ${isEven.test(3)}")

    // 11. Compose
    val f: (Int) -> Int = { it * 2 }
    val g: (Int) -> Int = { it + 3 }
    val fg = { x: Int -> g(f(x)) }  // g after f
    println("11. fg(5) = ${fg(5)}")  // (5*2)+3 = 13

    // 12. takeIf as lambda
    val result = (1..10).toList()
        .map { it.takeIf { n -> n % 3 == 0 } }
        .filterNotNull()
    println("12. divisible by 3: $result")

    // 13. let chain
    val value = " Hello "
        .let { it.trim() }
        .let { it.lowercase() }
        .let { "[$it]" }
    println("13. $value")

    // 14. Complex collection ops
    data class Item(val name: String, val value: Int, val active: Boolean)
    val items = listOf(
        Item("A", 10, true),
        Item("B", 20, false),
        Item("C", 30, true),
        Item("D", 40, true)
    )
    val total = items
        .filter { it.active }
        .map { it.value }
        .reduce { acc, v -> acc + v }
    println("14. active total = $total")

    println("\n=== Done! ===")
}
```

---

## สรุปส่วนที่ 10 (Summary)

| Concept | Syntax | ตัวอย่าง |
|---------|--------|---------|
| Lambda | `{ params -> body }` | `{ x -> x * 2 }` |
| Single param | `it` | `{ it * 2 }` |
| Function type | `(T) -> R` | `(Int) -> String` |
| Trailing lambda | `f() { ... }` | `list.map { it * 2 }` |
| Function ref | `::name` | `list.map(::println)` |
| `inline` | ลด overhead | `inline fun measure` |
| `noinline` | ป้องกัน inline | `noinline callback` |
| `crossinline` | ป้องกัน non-local return | `crossinline block` |
| `reified` | ใช้ type ใน inline | `reified T` |
| `fun interface` | SAM interface | `fun interface Pred` |

### เมื่อไหร่ควรใช้อะไร
- **Lambda**: callbacks, collection operations, event handlers
- **Function reference**: เมื่อมี named function ที่ต้องการ
- **inline**: HOF ที่เรียกบ่อย และต้องการ performance
- **SAM/fun interface**: Java interop หรือต้องการ name
- **Closure**: state ที่ต้องแชร์ระหว่าง callbacks

---

[← Part 09: Null Safety](../part09/README.md) | [Part 11: Extension Functions →](../part11/README.md)
