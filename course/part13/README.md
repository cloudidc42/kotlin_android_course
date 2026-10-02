# Part 13: Generics และ Type System
## ขั้นตอนที่ 276-300

---

## ขั้นตอนที่ 276: Generics คืออะไร?

Generics ช่วยให้เขียน code ที่ทำงานกับ type ต่างๆ ได้ โดยไม่ต้อง duplicate code

```kotlin
// ปัญหาโดยไม่มี Generics
fun printIntList(list: List<Int>) {
    list.forEach { println(it) }
}
fun printStringList(list: List<String>) {
    list.forEach { println(it) }
}

// แก้ด้วย Generics
fun <T> printList(list: List<T>) {
    list.forEach { println(it) }
}

// ใช้งาน
printList(listOf(1, 2, 3))         // T = Int
printList(listOf("a", "b", "c"))   // T = String
printList(listOf(1.5, 2.5, 3.5))   // T = Double
```

---

## ขั้นตอนที่ 277: Generic Classes

```kotlin
// Generic class พื้นฐาน
class Box<T>(val value: T) {
    fun get(): T = value
    override fun toString(): String = "Box($value)"
}

// ใช้งาน
val intBox = Box(42)
val strBox = Box("Hello")
val listBox = Box(listOf(1, 2, 3))

println(intBox.get())   // 42
println(strBox.get())   // Hello

// Multiple type parameters
class Pair<A, B>(val first: A, val second: B) {
    operator fun component1(): A = first
    operator fun component2(): B = second
    override fun toString() = "($first, $second)"
}

val pair = Pair("Kotlin", 2024)
val (lang, year) = pair
println("$lang - $year")  // Kotlin - 2024

// Generic class ที่ซับซ้อนขึ้น
class Stack<T> {
    private val items = mutableListOf<T>()
    
    fun push(item: T) { items.add(item) }
    fun pop(): T? = if (items.isEmpty()) null else items.removeAt(items.lastIndex)
    fun peek(): T? = items.lastOrNull()
    fun isEmpty(): Boolean = items.isEmpty()
    val size: Int get() = items.size
    
    override fun toString() = items.toString()
}

val stack = Stack<Int>()
stack.push(1)
stack.push(2)
stack.push(3)
println(stack.peek())   // 3
println(stack.pop())    // 3
println(stack.size)     // 2
```

---

## ขั้นตอนที่ 278: Upper Bounds (Type Constraints)

```kotlin
// T ต้องเป็น Number หรือ subclass
fun <T : Number> sum(list: List<T>): Double {
    return list.sumOf { it.toDouble() }
}

println(sum(listOf(1, 2, 3)))           // 6.0
println(sum(listOf(1.5, 2.5, 3.0)))    // 7.0
// sum(listOf("a", "b")) // Error! String ไม่ใช่ Number

// Multiple bounds ด้วย where
interface Printable { fun print() }
interface Serializable { fun serialize(): String }

fun <T> printAndSerialize(item: T)
    where T : Printable, T : Serializable {
    item.print()
    println(item.serialize())
}

// Comparable bound
fun <T : Comparable<T>> max(a: T, b: T): T {
    return if (a > b) a else b
}

println(max(3, 7))       // 7
println(max("apple", "banana"))  // banana
println(max(3.14, 2.71)) // 3.14

// Generic function กับ Comparable
fun <T : Comparable<T>> List<T>.minMax(): Pair<T, T>? {
    if (isEmpty()) return null
    var min = first()
    var max = first()
    forEach { item ->
        if (item < min) min = item
        if (item > max) max = item
    }
    return Pair(min, max)
}

val numbers = listOf(3, 1, 4, 1, 5, 9, 2, 6)
val (min, max) = numbers.minMax()!!
println("Min: $min, Max: $max")  // Min: 1, Max: 9
```

---

## ขั้นตอนที่ 279: Variance - in, out, *

```kotlin
// Covariant (out) - Producer
// อ่านได้อย่างเดียว
interface Producer<out T> {
    fun produce(): T
}

class StringProducer : Producer<String> {
    override fun produce(): String = "Hello"
}

// Producer<String> เป็น subtype ของ Producer<Any>
val producer: Producer<Any> = StringProducer()  // OK!
println(producer.produce())  // Hello

// Contravariant (in) - Consumer
// เขียนได้อย่างเดียว
interface Consumer<in T> {
    fun consume(item: T)
}

class AnyConsumer : Consumer<Any> {
    override fun consume(item: Any) { println("Consumed: $item") }
}

// Consumer<Any> เป็น subtype ของ Consumer<String>
val consumer: Consumer<String> = AnyConsumer()  // OK!
consumer.consume("Hello")

// Star Projection (*)
fun printList(list: List<*>) {
    list.forEach { println(it) }  // type เป็น Any?
}

printList(listOf(1, 2, 3))
printList(listOf("a", "b"))

// Invariant - ค่าเริ่มต้นสำหรับ class/interface
// MutableList<String> ไม่ใช่ subtype ของ MutableList<Any>
val strings: MutableList<String> = mutableListOf("a", "b")
// val anys: MutableList<Any> = strings  // Error!
val readOnly: List<Any> = strings  // OK เพราะ List เป็น covariant
```

---

## ขั้นตอนที่ 280: Reified Type Parameters

```kotlin
// ปัญหา: ปกติ type ถูก erased ตอน runtime
fun <T> isInstanceOf(value: Any): Boolean {
    // return value is T  // Error! Cannot check erased type
    return false
}

// แก้ด้วย reified (ใช้กับ inline function เท่านั้น)
inline fun <reified T> isInstanceOf(value: Any): Boolean {
    return value is T
}

println(isInstanceOf<String>("Hello"))  // true
println(isInstanceOf<Int>("Hello"))     // false
println(isInstanceOf<List<*>>(listOf(1,2,3)))  // true

// ใช้ reified กับ Gson/serialization
inline fun <reified T> String.fromJson(): T {
    return Gson().fromJson(this, T::class.java)
}

// ใช้ reified สำหรับ safe cast
inline fun <reified T> Any?.safeCast(): T? = this as? T

val value: Any = "Hello World"
val str: String? = value.safeCast()
val num: Int? = value.safeCast()
println(str)  // Hello World
println(num)  // null

// ใช้ reified เพื่อดึง KClass
inline fun <reified T> getClass(): KClass<T> = T::class

val stringClass = getClass<String>()
println(stringClass.simpleName)  // String
```

---

## ขั้นตอนที่ 281: Generic Functions ที่ใช้งานบ่อย

```kotlin
// Result wrapper
sealed class Result<out T> {
    data class Success<T>(val data: T) : Result<T>()
    data class Error(val exception: Throwable) : Result<Nothing>()
    object Loading : Result<Nothing>()
    
    val isSuccess get() = this is Success
    val isError get() = this is Error
    
    fun getOrNull(): T? = (this as? Success)?.data
    
    fun getOrElse(default: @UnsafeVariance T): T =
        (this as? Success)?.data ?: default
    
    fun getOrThrow(): T = when (this) {
        is Success -> data
        is Error -> throw exception
        else -> throw IllegalStateException("No value available")
    }
    
    fun <R> map(transform: (T) -> R): Result<R> = when (this) {
        is Success -> try { Success(transform(data)) } catch (e: Exception) { Error(e) }
        is Error -> Error(exception)
        else -> Loading
    }
    
    fun <R> flatMap(transform: (T) -> Result<R>): Result<R> = when (this) {
        is Success -> try { transform(data) } catch (e: Exception) { Error(e) }
        is Error -> Error(exception)
        else -> Loading
    }
    
    suspend fun <R> mapSuspend(transform: suspend (T) -> R): Result<R> = when (this) {
        is Success -> try { Success(transform(data)) } catch (e: Exception) { Error(e) }
        is Error -> Error(exception)
        else -> Loading
    }
}

// ใช้งาน
fun divideNumbers(a: Int, b: Int): Result<Double> {
    return if (b == 0) Result.Error(ArithmeticException("Division by zero"))
    else Result.Success(a.toDouble() / b)
}

val result = divideNumbers(10, 2)
    .map { it * 100 }
    .map { "Result: $it%" }

when (result) {
    is Result.Success -> println(result.data)   // Result: 500.0%
    is Result.Error -> println("Error: ${result.exception.message}")
    else -> println("Loading...")
}
```

---

## ขั้นตอนที่ 282: Generic Extensions และ Utility Functions

```kotlin
// Generic collection extensions
fun <T> List<T>.secondOrNull(): T? = if (size >= 2) this[1] else null
fun <T> List<T>.thirdOrNull(): T? = if (size >= 3) this[2] else null

fun <T, K> List<T>.groupByCount(keySelector: (T) -> K): Map<K, Int> {
    return groupBy(keySelector).mapValues { (_, v) -> v.size }
}

// ใช้งาน
data class Student(val name: String, val grade: String)
val students = listOf(
    Student("Alice", "A"),
    Student("Bob", "B"),
    Student("Charlie", "A"),
    Student("Dave", "C"),
    Student("Eve", "B")
)

val gradeCount = students.groupByCount { it.grade }
println(gradeCount)  // {A=2, B=2, C=1}

// Chunked processing
fun <T, R> List<T>.chunkedMap(chunkSize: Int, transform: (List<T>) -> List<R>): List<R> {
    return chunked(chunkSize).flatMap(transform)
}

// Safe index access
fun <T> List<T>.getOrNullSafe(index: Int): T? =
    if (index in indices) this[index] else null

// Swap in mutable list
fun <T> MutableList<T>.swap(i: Int, j: Int) {
    val temp = this[i]
    this[i] = this[j]
    this[j] = temp
}

val mutableList = mutableListOf(1, 2, 3, 4, 5)
mutableList.swap(0, 4)
println(mutableList)  // [5, 2, 3, 4, 1]

// Generic builder
class Builder<T>(private val factory: Builder<T>.() -> T) {
    private val steps = mutableListOf<T.() -> Unit>()
    
    fun step(block: T.() -> Unit): Builder<T> {
        steps.add(block)
        return this
    }
    
    fun build(): T {
        val item = factory()
        steps.forEach { it(item) }
        return item
    }
}
```

---

## ขั้นตอนที่ 283: Type Aliases

```kotlin
// typealias ช่วยให้อ่าน code ง่ายขึ้น
typealias UserList = List<User>
typealias UserMap = Map<String, User>
typealias Callback = () -> Unit
typealias ClickHandler = (view: View) -> Unit
typealias Predicate<T> = (T) -> Boolean
typealias Transformer<T, R> = (T) -> R

// Function type aliases
typealias OnSuccess<T> = (T) -> Unit
typealias OnError = (Throwable) -> Unit
typealias NetworkCallback<T> = (Result<T>) -> Unit

// ใช้งาน
fun fetchUsers(
    onSuccess: OnSuccess<UserList>,
    onError: OnError
) {
    // ...
}

// ใช้กับ Generic class
typealias StringResult = Result<String>
typealias IntResult = Result<Int>

fun parseNumber(str: String): IntResult {
    return try {
        Result.Success(str.toInt())
    } catch (e: NumberFormatException) {
        Result.Error(e)
    }
}

// Function aliases
typealias Validator<T> = (T) -> Boolean
typealias ValidatorWithMessage<T> = (T) -> Pair<Boolean, String>

val isNotEmpty: Validator<String> = { it.isNotEmpty() }
val isValidEmail: Validator<String> = { it.matches(Regex("^[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\\.[A-Za-z]{2,}$")) }

fun validate(
    value: String,
    validators: List<Validator<String>>
): Boolean = validators.all { it(value) }

println(validate("test@email.com", listOf(isNotEmpty, isValidEmail)))  // true
println(validate("", listOf(isNotEmpty, isValidEmail)))                // false
```

---

## ขั้นตอนที่ 284: Real-world Generic Patterns

```kotlin
// Repository Pattern ด้วย Generics
interface Repository<T, ID> {
    suspend fun findById(id: ID): T?
    suspend fun findAll(): List<T>
    suspend fun save(entity: T): T
    suspend fun delete(id: ID): Boolean
    suspend fun count(): Long
}

// Paginated Result
data class Page<T>(
    val items: List<T>,
    val page: Int,
    val pageSize: Int,
    val totalItems: Long,
    val totalPages: Int = ((totalItems + pageSize - 1) / pageSize).toInt()
) {
    val hasNext: Boolean get() = page < totalPages - 1
    val hasPrevious: Boolean get() = page > 0
    
    fun <R> map(transform: (T) -> R): Page<R> = Page(
        items = items.map(transform),
        page = page,
        pageSize = pageSize,
        totalItems = totalItems
    )
}

interface PageableRepository<T, ID> : Repository<T, ID> {
    suspend fun findAll(page: Int, pageSize: Int): Page<T>
    suspend fun search(query: String, page: Int = 0, pageSize: Int = 20): Page<T>
}

// Event System
typealias EventListener<T> = suspend (T) -> Unit

class EventBus<T> {
    private val listeners = mutableListOf<EventListener<T>>()
    
    fun subscribe(listener: EventListener<T>) {
        listeners.add(listener)
    }
    
    fun unsubscribe(listener: EventListener<T>) {
        listeners.remove(listener)
    }
    
    suspend fun emit(event: T) {
        listeners.forEach { it(event) }
    }
}

// Cache
class Cache<K, V>(private val maxSize: Int = 100) {
    private val map = LinkedHashMap<K, V>(maxSize, 0.75f, true)
    
    operator fun get(key: K): V? = map[key]
    
    operator fun set(key: K, value: V) {
        if (map.size >= maxSize) {
            map.remove(map.keys.first())
        }
        map[key] = value
    }
    
    fun getOrPut(key: K, default: () -> V): V {
        return map.getOrPut(key, default)
    }
    
    fun clear() = map.clear()
    val size: Int get() = map.size
}

// ใช้งาน Cache
val userCache = Cache<Long, User>(maxSize = 500)
userCache[1L] = User(1, "Alice", "alice@example.com")
val user = userCache[1L]
println(user?.name)  // Alice
```

---

## แบบฝึกหัด Part 13

```kotlin
// แบบฝึกหัดที่ 1: Generic Tree
// สร้าง Binary Search Tree

class BinarySearchTree<T : Comparable<T>> {
    private data class Node<T>(
        val value: T,
        var left: Node<T>? = null,
        var right: Node<T>? = null
    )
    
    private var root: Node<T>? = null
    
    fun insert(value: T) {
        root = insert(root, value)
    }
    
    private fun insert(node: Node<T>?, value: T): Node<T> {
        if (node == null) return Node(value)
        return when {
            value < node.value -> node.copy(left = insert(node.left, value))
            value > node.value -> node.copy(right = insert(node.right, value))
            else -> node
        }
    }
    
    fun contains(value: T): Boolean = contains(root, value)
    
    private fun contains(node: Node<T>?, value: T): Boolean {
        if (node == null) return false
        return when {
            value == node.value -> true
            value < node.value -> contains(node.left, value)
            else -> contains(node.right, value)
        }
    }
    
    fun toSortedList(): List<T> {
        val result = mutableListOf<T>()
        inorder(root, result)
        return result
    }
    
    private fun inorder(node: Node<T>?, result: MutableList<T>) {
        if (node == null) return
        inorder(node.left, result)
        result.add(node.value)
        inorder(node.right, result)
    }
}

fun main() {
    val bst = BinarySearchTree<Int>()
    listOf(5, 3, 7, 1, 4, 6, 8).forEach { bst.insert(it) }
    println(bst.toSortedList())  // [1, 3, 4, 5, 6, 7, 8]
    println(bst.contains(4))    // true
    println(bst.contains(9))    // false
}
```

---

*Part 13 จบแล้ว | ก่อนหน้า: [Part 12](../part12/README.md) | ถัดไป: [Part 14](../part14/README.md)*
