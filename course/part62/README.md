# Part 62: Kotlin Compiler & Advanced Language Features
## ขั้นตอนที่ 1026-1050

---

## ขั้นตอนที่ 1026: Kotlin Compiler Internals

```
Kotlin Compilation Process:

Source (.kt) → Frontend → IR → Backend → Output

1. Lexer/Parser: text → AST (Abstract Syntax Tree)
2. Type Resolution: ตรวจสอบ types
3. Frontend Analysis: null checks, smart casts
4. IR Generation: Platform-independent IR
5. Backend:
   - JVM: IR → JVM Bytecode (.class)
   - JS: IR → JavaScript
   - Native: IR → LLVM IR → Native binary

Kotlin Compiler Plugins:
- kotlinx.serialization
- Jetpack Compose (Compose Compiler Plugin)
- AllOpen, NoArg
- Custom plugins
```

---

## ขั้นตอนที่ 1027: Inline Classes (Value Classes)

```kotlin
// ============================================
// Value Classes - type safety without overhead
// ============================================

// ปัญหา: primitive types ไม่มี type safety
fun processOrder(userId: Long, orderId: Long, productId: Long) {
    // ❌ ง่ายต่อการส่งค่าผิดลำดับ
}

// ✅ Value classes - zero overhead wrappers
@JvmInline
value class UserId(val value: Long)

@JvmInline
value class OrderId(val value: Long)

@JvmInline
value class ProductId(val value: Long)

fun processOrder(userId: UserId, orderId: OrderId, productId: ProductId) {
    // ✅ compiler จะจับ type mismatch
}

// ใช้งาน
processOrder(
    userId = UserId(1L),
    orderId = OrderId(42L),
    productId = ProductId(100L)
)

// ============================================
// Value class สำหรับ units
// ============================================

@JvmInline
value class Meters(val value: Double) {
    operator fun plus(other: Meters) = Meters(value + other.value)
    operator fun times(factor: Double) = Meters(value * factor)
    val inKilometers get() = value / 1000.0
}

@JvmInline
value class Kilograms(val value: Double) {
    val inGrams get() = value * 1000.0
    val inPounds get() = value * 2.20462
}

fun calculateShipping(distance: Meters, weight: Kilograms): Double {
    return distance.value * 0.001 + weight.value * 0.5
}

val dist = Meters(5000.0)
val weight = Kilograms(2.5)
val cost = calculateShipping(dist, weight)

// ============================================
// String value classes
// ============================================

@JvmInline
value class Email(val value: String) {
    init {
        require(value.contains("@")) { "Invalid email: $value" }
    }
}

@JvmInline
value class PasswordHash(private val hash: String) {
    fun matches(plaintext: String): Boolean {
        return BCrypt.checkpw(plaintext, hash)
    }
}
```

---

## ขั้นตอนที่ 1028: Operator Overloading ขั้นสูง

```kotlin
// ============================================
// Operator Overloading
// ============================================

data class Matrix(val rows: Int, val cols: Int, val data: Array<FloatArray>) {
    
    // +
    operator fun plus(other: Matrix): Matrix {
        require(rows == other.rows && cols == other.cols) { "Size mismatch" }
        return Matrix(rows, cols, Array(rows) { r ->
            FloatArray(cols) { c -> data[r][c] + other.data[r][c] }
        })
    }
    
    // *
    operator fun times(other: Matrix): Matrix {
        require(cols == other.rows) { "Cannot multiply ${rows}x$cols by ${other.rows}x${other.cols}" }
        return Matrix(rows, other.cols, Array(rows) { r ->
            FloatArray(other.cols) { c ->
                (0 until cols).sumOf { k -> (data[r][k] * other.data[k][c]).toDouble() }.toFloat()
            }
        })
    }
    
    // scalar multiplication
    operator fun times(scalar: Float): Matrix {
        return Matrix(rows, cols, Array(rows) { r ->
            FloatArray(cols) { c -> data[r][c] * scalar }
        })
    }
    
    // indexing
    operator fun get(row: Int, col: Int): Float = data[row][col]
    operator fun set(row: Int, col: Int, value: Float) { data[row][col] = value }
    
    // unary minus (negate)
    operator fun unaryMinus(): Matrix = times(-1f)
    
    companion object {
        fun identity(n: Int) = Matrix(n, n, Array(n) { r ->
            FloatArray(n) { c -> if (r == c) 1f else 0f }
        })
    }
}

// ============================================
// Comparable
// ============================================

data class Version(val major: Int, val minor: Int, val patch: Int) : Comparable<Version> {
    override fun compareTo(other: Version): Int {
        val majorDiff = major.compareTo(other.major)
        if (majorDiff != 0) return majorDiff
        
        val minorDiff = minor.compareTo(other.minor)
        if (minorDiff != 0) return minorDiff
        
        return patch.compareTo(other.patch)
    }
    
    override fun toString() = "$major.$minor.$patch"
    
    companion object {
        fun parse(s: String): Version {
            val (major, minor, patch) = s.split(".").map { it.toInt() }
            return Version(major, minor, patch)
        }
    }
}

val v1 = Version.parse("1.2.3")
val v2 = Version.parse("1.3.0")
println(v1 < v2)   // true
println(maxOf(v1, v2))  // 1.3.0

val versions = listOf(Version(2,0,0), Version(1,2,3), Version(1,10,0))
println(versions.sorted())  // [1.2.3, 1.10.0, 2.0.0]
```

---

## ขั้นตอนที่ 1029: Kotlin Contracts

```kotlin
// ============================================
// Contracts - ให้ compiler รู้ข้อมูลเพิ่มเติม
// ============================================

// ใช้ @OptIn(ExperimentalContracts::class)

// ตัวอย่าง: ถ้า function return true แล้ว parameter ไม่เป็น null
fun isNotNull(value: Any?): Boolean {
    contract {
        returns(true) implies (value != null)
    }
    return value != null
}

fun test(x: String?) {
    if (isNotNull(x)) {
        println(x.length)  // ✅ compiler รู้ว่า x != null
    }
}

// ============================================
// callsInPlace - บอก compiler ว่า lambda ถูกเรียกกี่ครั้ง
// ============================================

inline fun <T> runExactlyOnce(block: () -> T): T {
    contract {
        callsInPlace(block, InvocationKind.EXACTLY_ONCE)
    }
    return block()
}

fun example() {
    val name: String
    runExactlyOnce {
        name = "Alice"  // ✅ compiler ยอมรับ definite assignment
    }
    println(name)  // ✅ ไม่ error: name guaranteed initialized
}

// ตัวอย่าง: require/check ใน standard library ก็ใช้ contracts
fun process(value: String?) {
    requireNotNull(value) { "Value must not be null" }
    println(value.length)  // ✅ smart cast เพราะ contract
}
```

---

## ขั้นตอนที่ 1030: Reflection

```kotlin
// ============================================
// Kotlin Reflection
// ============================================

import kotlin.reflect.*
import kotlin.reflect.full.*

data class User(val id: Long, val name: String, val email: String, val age: Int)

fun inspectClass() {
    val klass = User::class
    
    println("Class: ${klass.simpleName}")
    println("Is data class: ${klass.isData}")
    println("Is abstract: ${klass.isAbstract}")
    
    // Properties
    klass.memberProperties.forEach { prop ->
        println("Property: ${prop.name}: ${prop.returnType}")
    }
    
    // Constructors
    klass.constructors.forEach { ctor ->
        println("Constructor params: ${ctor.parameters.map { it.name }}")
    }
    
    // Primary constructor
    val primaryCtor = klass.primaryConstructor
    println("Primary ctor: ${primaryCtor?.parameters?.map { "${it.name}: ${it.type}" }}")
}

// Dynamic instantiation
fun createFromMap(klass: KClass<*>, data: Map<String, Any>): Any? {
    val ctor = klass.primaryConstructor ?: return null
    
    val args = ctor.parameters.associateWith { param ->
        data[param.name]
    }
    
    return ctor.callBy(args)
}

// ใช้สำหรับ JSON mapping
val user = createFromMap(
    User::class,
    mapOf("id" to 1L, "name" to "Alice", "email" to "alice@ex.com", "age" to 25)
) as User

// Generic type info
inline fun <reified T> getGenericType(): KType = typeOf<T>()

val listType = getGenericType<List<String>>()
println(listType)  // kotlin.collections.List<kotlin.String>
```

---

## แบบฝึกหัด Part 62

```kotlin
// แบบฝึกหัด: สร้าง mini ORM ด้วย Reflection

@Target(AnnotationTarget.CLASS)
annotation class Table(val name: String)

@Target(AnnotationTarget.PROPERTY)
annotation class Column(val name: String = "", val primaryKey: Boolean = false)

@Table("users")
data class UserEntity(
    @Column(primaryKey = true) val id: Long,
    @Column("user_name") val name: String,
    @Column val email: String
)

// TODO: สร้าง ORM functions ด้วย Reflection:
object MiniOrm {
    
    fun <T : Any> getTableName(klass: KClass<T>): String {
        // TODO: อ่าน @Table annotation
        return ""
    }
    
    fun <T : Any> getInsertSql(entity: T): String {
        // TODO: สร้าง INSERT SQL จาก @Column annotations
        // INSERT INTO users (id, user_name, email) VALUES (?, ?, ?)
        return ""
    }
    
    fun <T : Any> getSelectAllSql(klass: KClass<T>): String {
        // TODO: สร้าง SELECT * SQL
        return ""
    }
    
    fun <T : Any> mapResultToEntity(result: Map<String, Any>, klass: KClass<T>): T {
        // TODO: map column values กลับเป็น entity
        TODO()
    }
}

// Test
println(MiniOrm.getTableName(UserEntity::class))  // "users"
println(MiniOrm.getInsertSql(UserEntity(1, "Alice", "alice@ex.com")))
// INSERT INTO users (id, user_name, email) VALUES (1, 'Alice', 'alice@ex.com')
```

---

*Part 62 จบแล้ว | ก่อนหน้า: [Part 61](../part61/README.md) | ถัดไป: [Part 63](../part63/README.md)*
