# Part 09: Null Safety and Elvis Operator

## ภาพรวม (Overview)
ในส่วนนี้เราจะเรียนรู้เกี่ยวกับ Null Safety ซึ่งเป็นหนึ่งในคุณสมบัติที่ยอดเยี่ยมที่สุดของ Kotlin
ช่วยป้องกัน NullPointerException ที่เป็นปัญหาใหญ่ใน Java

**Steps ที่ครอบคลุม:** 176-200

---

## Step 176: Nullable Types - ประเภทที่รับ null ได้

```kotlin
fun main() {
    // Non-nullable types - ไม่สามารถเป็น null ได้
    var name: String = "สมชาย"
    // name = null  // Error! ไม่สามารถเป็น null

    // Nullable types - ใช้ ? ต่อท้าย type
    var nullableName: String? = "สมหญิง"
    nullableName = null  // OK

    var age: Int? = 25
    age = null  // OK

    println("name: $name")
    println("nullableName: $nullableName")
    println("age: $age")

    // Nullable vs Non-nullable
    val nonNull: String = "hello"
    val nullable: String? = null

    // การใช้งานที่แตกต่างกัน
    println(nonNull.length)  // ทำได้ทันที
    // println(nullable.length)  // Error! ต้องตรวจสอบ null ก่อน

    // ต้องตรวจสอบก่อน
    if (nullable != null) {
        println(nullable.length)  // Smart cast: ใน if block นี้ nullable เป็น String (not nullable)
    }

    // Nullable collections
    val list: List<String>? = null
    val listWithNulls: List<String?> = listOf("a", null, "b", null, "c")
    val nullableListWithNulls: List<String?>? = null

    println("\nnullableListWithNulls: $nullableListWithNulls")
    println("listWithNulls: $listWithNulls")

    // ตัวอย่างการใช้ nullable ใน function
    fun findUser(id: Int): String? {
        val users = mapOf(1 to "สมชาย", 2 to "สมหญิง")
        return users[id]  // Map.get() คืน V? (nullable)
    }

    println("\nfindUser(1): ${findUser(1)}")
    println("findUser(99): ${findUser(99)}")

    // Nullable types ใน data class
    data class Person(
        val name: String,
        val nickname: String?,
        val age: Int,
        val email: String?
    )

    val person1 = Person("สมชาย", "ชาย", 25, "somchai@email.com")
    val person2 = Person("สมหญิง", null, 30, null)

    listOf(person1, person2).forEach { person ->
        val display = person.nickname ?: person.name
        println("$display, email=${person.email ?: "ไม่มี"}")
    }
}
```

---

## Step 177: Safe Call Operator ?.

```kotlin
data class Company(val name: String, val address: Address?)
data class Address(val street: String, val city: String, val country: Country?)
data class Country(val name: String, val code: String)

fun main() {
    val company1 = Company("TechCorp", Address("123 Main St", "กรุงเทพ", Country("Thailand", "TH")))
    val company2 = Company("StartupXYZ", Address("456 Second St", "เชียงใหม่", null))
    val company3 = Company("GhostCo", null)

    // Safe call ?.
    // ถ้าซ้ายมือเป็น null จะคืน null แทนการ throw NPE
    println(company1.address?.city)          // กรุงเทพ
    println(company2.address?.country?.code) // null
    println(company3.address?.city)          // null

    // Chain safe calls
    println("\n--- Chained safe calls ---")
    val country1 = company1.address?.country?.name
    val country2 = company2.address?.country?.name
    val country3 = company3.address?.country?.name

    println("company1 country: $country1")  // Thailand
    println("company2 country: $country2")  // null
    println("company3 country: $country3")  // null

    // Safe call กับ method
    val nullString: String? = null
    val nonNullString: String? = "Hello World"

    println("\n--- Safe call with methods ---")
    println(nullString?.length)          // null
    println(nonNullString?.length)       // 11
    println(nullString?.uppercase())     // null
    println(nonNullString?.uppercase())  // HELLO WORLD

    // Safe call กับ extension function
    println(nullString?.reversed())      // null
    println(nonNullString?.reversed())   // dlroW olleH

    // let กับ safe call
    println("\n--- let with safe call ---")
    nullString?.let {
        println("this won't print: $it")
    }

    nonNullString?.let {
        println("this will print: $it (length=${it.length})")
    }

    // ใช้ใน collection
    println("\n--- Safe calls in collections ---")
    val companies = listOf(company1, company2, company3)
    
    companies.forEach { company ->
        val cityInfo = company.address?.city ?: "ไม่ระบุเมือง"
        println("${company.name}: $cityInfo")
    }

    // mapNotNull กับ safe calls
    val countryCodes = companies.mapNotNull { it.address?.country?.code }
    println("Country codes: $countryCodes")
}
```

---

## Step 178: Elvis Operator ?:

```kotlin
data class User(val name: String, val bio: String?, val followersCount: Int?)

fun getGreeting(name: String?): String {
    // Elvis operator: ถ้าซ้ายเป็น null ใช้ค่าขวาแทน
    return "สวัสดี ${name ?: "ผู้เยี่ยมชม"}!"
}

fun getAge(age: Int?): Int {
    return age ?: 0  // ถ้า null ใช้ 0
}

fun main() {
    println(getGreeting("สมชาย"))
    println(getGreeting(null))
    println("อายุ: ${getAge(25)}")
    println("อายุ: ${getAge(null)}")

    val users = listOf(
        User("สมชาย", "นักพัฒนา Kotlin", 1500),
        User("สมหญิง", null, null),
        User("สมศักดิ์", "Designer", 500)
    )

    users.forEach { user ->
        val bio = user.bio ?: "ยังไม่มีประวัติ"
        val followers = user.followersCount ?: 0
        println("${user.name}: $bio ($followers followers)")
    }

    // Elvis กับ throw
    println("\n--- Elvis with throw ---")
    fun getUserById(id: Int, users: Map<Int, String>): String {
        return users[id] ?: throw IllegalArgumentException("ไม่พบ user id=$id")
    }

    val userMap = mapOf(1 to "สมชาย", 2 to "สมหญิง")
    
    println(getUserById(1, userMap))
    try {
        println(getUserById(99, userMap))
    } catch (e: IllegalArgumentException) {
        println("Error: ${e.message}")
    }

    // Elvis กับ return
    fun processUser(user: User?): String {
        val u = user ?: return "null user"
        val name = u.name
        val bio = u.bio ?: return "no bio for $name"
        return "$name: $bio"
    }

    println("\n${processUser(users[0])}")
    println(processUser(users[1]))
    println(processUser(null))

    // Nested Elvis
    val map: Map<String, String?> = mapOf("key1" to "value1", "key2" to null)
    
    val v1 = map["key1"] ?: "default"
    val v2 = map["key2"] ?: "default"
    val v3 = map["key3"] ?: "default"
    
    println("\n$v1, $v2, $v3")

    // Elvis กับ let
    val maybeNull: String? = null
    val result = maybeNull?.let { it.length } ?: -1
    println("result: $result")
    
    val notNull: String? = "kotlin"
    val result2 = notNull?.let { it.length } ?: -1
    println("result2: $result2")
}
```

---

## Step 179: Non-null Assertion !!

```kotlin
fun main() {
    // !! operator - บอกว่า "ฉันแน่ใจว่าไม่ใช่ null"
    // ถ้าเป็น null จะ throw KotlinNullPointerException

    var name: String? = "สมชาย"
    val length = name!!.length  // OK เพราะ name มีค่า
    println("length: $length")

    // อันตราย!
    name = null
    try {
        val len = name!!.length  // throw NullPointerException
        println("จะไม่ถึงตรงนี้")
    } catch (e: NullPointerException) {
        println("NullPointerException! อย่าใช้ !! กับ null")
    }

    // เมื่อไหร่ที่ !! เหมาะสม?
    // 1. เมื่อรู้แน่ว่าไม่ null แต่ compiler ไม่รู้
    
    val configs = mapOf("db.host" to "localhost", "db.port" to "5432")
    val requiredKeys = listOf("db.host", "db.port")
    
    // เราตรวจสอบแล้วว่า keys มีอยู่ก่อน
    val allPresent = requiredKeys.all { it in configs }
    if (allPresent) {
        // !!  เหมาะสมตรงนี้เพราะเราตรวจสอบแล้ว
        val host = configs["db.host"]!!
        val port = configs["db.port"]!!.toInt()
        println("DB: $host:$port")
    }

    // 2. ใน test code
    data class Result(val value: String?)
    
    fun fetchResult(): Result = Result("test-value")
    
    val result = fetchResult()
    // ใน test เราอาจ assert ว่าไม่ null
    val value = result.value!!
    println("value: $value")

    // ทางเลือกที่ดีกว่า !! ในส่วนใหญ่
    val nullable: String? = null
    
    // แทนที่จะใช้ !!
    // val bad = nullable!!  // อันตราย
    
    // ใช้ ?: แทน
    val good1 = nullable ?: "default"
    
    // ใช้ requireNotNull
    try {
        val good2 = requireNotNull(nullable) { "ค่าต้องไม่เป็น null" }
    } catch (e: IllegalArgumentException) {
        println("requireNotNull: ${e.message}")
    }
    
    // ใช้ checkNotNull
    try {
        val good3 = checkNotNull(nullable) { "check failed" }
    } catch (e: IllegalStateException) {
        println("checkNotNull: ${e.message}")
    }
    
    // ใช้ let
    nullable?.let { value ->
        println("has value: $value")
    } ?: println("no value")
}
```

---

## Step 180: Smart Cast

```kotlin
open class Animal(val name: String)
class Dog(name: String, val breed: String) : Animal(name) {
    fun bark() = println("$name: Woof!")
}
class Cat(name: String, val isIndoor: Boolean) : Animal(name) {
    fun purr() = println("$name: Purrrr~")
}

fun main() {
    // Smart Cast - Kotlin อนุมาน type หลัง null check หรือ is check

    // 1. Null check smart cast
    var text: String? = "Hello Kotlin"
    
    if (text != null) {
        // text ถูก smart cast เป็น String (non-nullable) ใน block นี้
        println("Length: ${text.length}")
        println("Upper: ${text.uppercase()}")
    }

    // 2. is check smart cast
    val animals: List<Animal> = listOf(
        Dog("บัดดี้", "Labrador"),
        Cat("วิสกี้", true),
        Dog("แม็กซ์", "Poodle"),
        Animal("นกแก้ว")
    )

    for (animal in animals) {
        when (animal) {
            is Dog -> {
                // animal ถูก smart cast เป็น Dog ใน branch นี้
                animal.bark()
                println("  Breed: ${animal.breed}")
            }
            is Cat -> {
                // animal ถูก smart cast เป็น Cat ใน branch นี้
                animal.purr()
                println("  Indoor: ${animal.isIndoor}")
            }
            else -> println("${animal.name}: เป็นสัตว์ทั่วไป")
        }
    }

    // 3. Smart cast ใน when expression
    fun describe(obj: Any): String {
        return when (obj) {
            is Int -> "จำนวนเต็ม: $obj (${if (obj > 0) "บวก" else if (obj < 0) "ลบ" else "ศูนย์"})"
            is Double -> "ทศนิยม: $obj"
            is String -> "ข้อความ: '$obj' (${obj.length} ตัวอักษร)"
            is List<*> -> "List ขนาด ${obj.size}"
            is Boolean -> "Boolean: $obj"
            null -> "null"
            else -> "ไม่รู้จัก: ${obj::class.simpleName}"
        }
    }

    println("\n--- Smart cast with Any ---")
    listOf(42, 3.14, "Hello", listOf(1, 2, 3), true, null, Any()).forEach {
        println(describe(it))
    }

    // 4. Smart cast เงื่อนไข
    fun processString(s: String?): Int {
        if (s == null || s.isBlank()) return 0
        // Smart cast: s เป็น String (non-nullable) และไม่ blank
        return s.trim().length
    }

    println("\n--- processString ---")
    println(processString("  Hello  "))
    println(processString(""))
    println(processString(null))

    // 5. Smart cast ไม่ทำงานเมื่อ var และ multi-thread
    var mutableText: String? = "test"
    // compiler ไม่แน่ใจว่า thread อื่นจะเปลี่ยนค่าหรือไม่
    // ต้องกำหนด local variable ก่อน
    val localText = mutableText
    if (localText != null) {
        println("local: ${localText.length}")  // works
    }
}
```

---

## Step 181: let กับ Null Safety

```kotlin
data class UserProfile(
    val username: String,
    val email: String?,
    val phone: String?,
    val avatar: String?
)

fun sendEmail(to: String, subject: String) {
    println("ส่ง email ถึง $to: $subject")
}

fun sendSms(phone: String, message: String) {
    println("ส่ง SMS ถึง $phone: $message")
}

fun loadAvatar(url: String): String {
    return "Image[$url]"
}

fun main() {
    val user = UserProfile(
        username = "somchai",
        email = "somchai@example.com",
        phone = null,
        avatar = "https://example.com/avatar.jpg"
    )

    val userNoContact = UserProfile(
        username = "anonymous",
        email = null,
        phone = null,
        avatar = null
    )

    // let ทำงานเมื่อ non-null
    println("--- let ---")
    user.email?.let { email ->
        sendEmail(email, "ยินดีต้อนรับ!")
    }
    
    user.phone?.let { phone ->
        sendSms(phone, "ยืนยัน OTP")
    } ?: println("ไม่มีเบอร์โทร ไม่ส่ง SMS")

    user.avatar?.let { url ->
        val image = loadAvatar(url)
        println("โหลด avatar: $image")
    }

    println()

    // userNoContact
    userNoContact.email?.let {
        sendEmail(it, "ข้อความ")
    } ?: println("${userNoContact.username}: ไม่มี email")

    userNoContact.avatar?.let {
        loadAvatar(it)
    } ?: println("${userNoContact.username}: ไม่มี avatar")

    // run กับ nullable
    println("\n--- run ---")
    val result = user.email?.run {
        val domain = substringAfter("@")
        "Email domain: $domain"
    }
    println(result ?: "No email")

    // also กับ nullable
    println("\n--- also ---")
    user.avatar?.also { url ->
        println("กำลัง cache URL: $url")
    }?.let { url ->
        loadAvatar(url)
    }.also { image ->
        println("Image loaded: $image")
    }

    // apply กับ nullable
    println("\n--- apply ---")
    data class Config(var host: String = "", var port: Int = 0)
    
    val configStr: String? = "localhost:8080"
    val config = configStr?.let { str ->
        Config().apply {
            host = str.substringBefore(":")
            port = str.substringAfter(":").toInt()
        }
    }
    println("Config: $config")
    
    val nullConfigStr: String? = null
    val nullConfig = nullConfigStr?.let { str ->
        Config().apply {
            host = str.substringBefore(":")
        }
    }
    println("nullConfig: $nullConfig")
    
    // takeIf และ takeUnless
    println("\n--- takeIf/takeUnless ---")
    val number = 42
    
    val evenNumber = number.takeIf { it % 2 == 0 }
    println("even: $evenNumber")  // 42
    
    val oddNumber = number.takeIf { it % 2 != 0 }
    println("odd: $oddNumber")  // null
    
    val notLarge = number.takeUnless { it > 100 }
    println("notLarge: $notLarge")  // 42
    
    val notSmall = number.takeUnless { it < 100 }
    println("notSmall: $notSmall")  // null
    
    // takeIf กับ null safety
    val users = listOf("สมชาย", "สมหญิง", "", "สมศักดิ์", null, "สมใจ")
    val validNames = users.mapNotNull { it?.takeIf { name -> name.isNotBlank() } }
    println("\nValid names: $validNames")
}
```

---

## Step 182: Safe Cast as?

```kotlin
open class Vehicle(val brand: String)
class Car(brand: String, val doors: Int) : Vehicle(brand)
class Motorcycle(brand: String, val hasSidecar: Boolean) : Vehicle(brand)
class Truck(brand: String, val payload: Double) : Vehicle(brand)

fun main() {
    // as? - Safe cast ถ้า cast ไม่ได้จะคืน null แทน throw exception

    val vehicles: List<Vehicle> = listOf(
        Car("Toyota", 4),
        Motorcycle("Honda", false),
        Truck("Isuzu", 5.0),
        Car("BMW", 2),
        Motorcycle("Harley", true)
    )

    println("--- as? Safe Cast ---")
    for (vehicle in vehicles) {
        val car = vehicle as? Car
        val moto = vehicle as? Motorcycle
        val truck = vehicle as? Truck
        
        when {
            car != null -> println("${car.brand}: ${car.doors}-door Car")
            moto != null -> println("${moto.brand}: Motorcycle (sidecar: ${moto.hasSidecar})")
            truck != null -> println("${truck.brand}: Truck (payload: ${truck.payload}t)")
        }
    }

    // ต่างจาก as
    println("\n--- as vs as? ---")
    val obj: Any = "Hello"
    
    val str1 = obj as String  // OK
    println("str1: $str1")
    
    val str2 = obj as? String  // OK
    println("str2: $str2")
    
    val num1 = obj as? Int  // null (ไม่ crash)
    println("num1: $num1")
    
    try {
        val num2 = obj as Int  // throw ClassCastException
    } catch (e: ClassCastException) {
        println("as Int crashed: ${e.message}")
    }

    // ใช้ประโยชน์จาก as?
    println("\n--- Practical as? ---")
    
    // กรองและ cast พร้อมกัน
    val cars = vehicles.filterIsInstance<Car>()
    println("Cars: ${cars.map { it.brand }}")
    
    // as? กับ smart cast
    fun processVehicle(v: Any?): String {
        return when (val typed = v as? Vehicle) {
            null -> "ไม่ใช่ยานพาหนะ"
            is Car -> "${typed.brand} Car with ${typed.doors} doors"
            is Truck -> "${typed.brand} Truck"
            else -> "${typed.brand} Vehicle"
        }
    }
    
    println(processVehicle(vehicles[0]))
    println(processVehicle("not a vehicle"))
    println(processVehicle(null))
    
    // JSON parsing แบบปลอดภัย
    println("\n--- Safe JSON-like parsing ---")
    val jsonData: Map<String, Any?> = mapOf(
        "name" to "สมชาย",
        "age" to 25,
        "score" to 85.5,
        "active" to true,
        "address" to null
    )
    
    val name = jsonData["name"] as? String ?: "unknown"
    val age = jsonData["age"] as? Int ?: 0
    val score = jsonData["score"] as? Double ?: 0.0
    val active = jsonData["active"] as? Boolean ?: false
    val address = jsonData["address"] as? String
    val missing = jsonData["missing"] as? String
    
    println("name: $name, age: $age, score: $score, active: $active")
    println("address: $address, missing: $missing")
}
```

---

## Step 183: Null Checks Best Practices

```kotlin
data class OrderItem(val productId: Int, val name: String, val price: Double, val qty: Int)
data class Order(
    val id: Int,
    val customerId: Int,
    val items: List<OrderItem>?,
    val discount: Double?,
    val promoCode: String?
)

fun calculateTotal(order: Order): Double {
    val items = order.items ?: return 0.0
    
    val subtotal = items.sumOf { it.price * it.qty }
    val discount = order.discount ?: 0.0
    
    return subtotal * (1 - discount)
}

fun applyPromoCode(code: String?): Double {
    return when (code?.uppercase()) {
        "SAVE10" -> 0.10
        "SAVE20" -> 0.20
        "HALFOFF" -> 0.50
        null -> 0.0
        else -> 0.0
    }
}

fun main() {
    // Best Practices

    // 1. ใช้ Elvis กับ early return
    println("--- Early return ---")
    fun processOrder(order: Order?) {
        val o = order ?: run {
            println("ไม่มี order")
            return
        }
        
        val items = o.items ?: run {
            println("Order ${o.id} ไม่มีสินค้า")
            return
        }
        
        println("Order ${o.id}: ${items.size} รายการ")
        println("ยอดรวม: ${calculateTotal(o)}")
    }

    val order1 = Order(1, 101, listOf(
        OrderItem(1, "iPhone", 35000.0, 1),
        OrderItem(2, "Case", 500.0, 2)
    ), 0.1, "SAVE10")
    
    val order2 = Order(2, 102, null, null, null)
    
    processOrder(order1)
    processOrder(order2)
    processOrder(null)

    // 2. ใช้ let สำหรับ single action กับ nullable
    println("\n--- let for single action ---")
    val promoDiscount = order1.promoCode?.let { code ->
        val discount = applyPromoCode(code)
        "Promo $code: -${(discount * 100).toInt()}%"
    } ?: "ไม่มีโปรโมชัน"
    println(promoDiscount)

    // 3. ใช้ when กับ nullable
    println("\n--- when with nullable ---")
    fun describeDiscount(discount: Double?): String {
        return when {
            discount == null -> "ไม่มีส่วนลด"
            discount == 0.0 -> "ส่วนลด 0%"
            discount < 0.1 -> "ส่วนลดน้อย (${(discount*100).toInt()}%)"
            discount < 0.3 -> "ส่วนลดปานกลาง (${(discount*100).toInt()}%)"
            else -> "ส่วนลดมาก (${(discount*100).toInt()}%)"
        }
    }
    
    listOf(null, 0.0, 0.05, 0.2, 0.5).forEach { discount ->
        println(describeDiscount(discount))
    }

    // 4. ใช้ requireNotNull และ checkNotNull
    println("\n--- requireNotNull ---")
    fun processPayment(amount: Double, currency: String?) {
        val curr = requireNotNull(currency) { "currency ต้องไม่เป็น null" }
        println("ชำระ $amount $curr")
    }
    
    processPayment(1000.0, "THB")
    try {
        processPayment(1000.0, null)
    } catch (e: IllegalArgumentException) {
        println("Error: ${e.message}")
    }

    // 5. Collection null handling
    println("\n--- Collection null handling ---")
    val orders = listOf(order1, order2)
    
    // filterNotNull ลบ null items
    val allItems = orders
        .mapNotNull { it.items }
        .flatten()
    println("สินค้าทั้งหมด: ${allItems.map { it.name }}")
    
    // orEmpty() สำหรับ nullable collection
    orders.forEach { order ->
        val items = order.items.orEmpty()
        println("Order ${order.id}: ${items.size} items")
    }
}
```

---

## Step 184: Platform Types และ Null Safety กับ Java

```kotlin
// ใน Kotlin เมื่อเรียก Java code ที่ไม่มี @Nullable/@NonNull annotations
// จะได้ Platform Type (String!) ซึ่งไม่ทราบว่า nullable หรือไม่

// จำลอง Java class
class JavaLikeCode {
    // เหมือน Java method ที่อาจคืน null
    fun mayReturnNull(value: Int): String? {
        return if (value > 0) "positive: $value" else null
    }
    
    // เหมือน Java method ที่ไม่คืน null
    fun neverNull(value: Int): String {
        return "value: $value"
    }
}

// Null safety annotations
// ใน Kotlin เราสามารถใช้:
// @Nullable - บอกว่า nullable
// @NonNull / @NotNull - บอกว่าไม่ null

fun main() {
    val javaCode = JavaLikeCode()
    
    // เรียก method ที่อาจ null
    val result1 = javaCode.mayReturnNull(5)
    val result2 = javaCode.mayReturnNull(-1)
    
    println("result1: $result1")
    println("result2: $result2")
    
    // ปลอดภัย
    result1?.let { println("result1 length: ${it.length}") }
    result2?.let { println("result2 length: ${it.length}") } 
        ?: println("result2 is null")

    // Defensive programming
    println("\n--- Defensive Patterns ---")
    
    // Pattern 1: ตรวจสอบก่อนใช้
    fun safeLength(s: String?): Int {
        return s?.length ?: 0
    }
    
    // Pattern 2: Default value
    fun safeGet(map: Map<String, String?>, key: String, default: String = ""): String {
        return map[key] ?: default
    }
    
    val testMap = mapOf("a" to "hello", "b" to null, "c" to "world")
    println(safeGet(testMap, "a"))        // hello
    println(safeGet(testMap, "b"))        // (empty)
    println(safeGet(testMap, "d", "N/A")) // N/A

    // Pattern 3: fail fast
    fun validateAndProcess(data: String?) {
        val d = data ?: throw IllegalArgumentException("data ต้องไม่เป็น null")
        require(d.isNotBlank()) { "data ต้องไม่ว่าง" }
        println("Processing: $d")
    }
    
    validateAndProcess("Hello")
    
    listOf("World", null, "", "Kotlin").forEach { input ->
        try {
            validateAndProcess(input)
        } catch (e: Exception) {
            println("Error for '$input': ${e.message}")
        }
    }
    
    // Pattern 4: Null object pattern
    println("\n--- Null Object Pattern ---")
    
    interface Notification {
        fun send(message: String)
    }
    
    class EmailNotification(val email: String) : Notification {
        override fun send(message: String) = println("Email to $email: $message")
    }
    
    // Null Object - ทำอะไรก็ไม่เกิดผล
    object NoNotification : Notification {
        override fun send(message: String) {}  // do nothing
    }
    
    fun getNotification(email: String?): Notification {
        return if (email != null) EmailNotification(email) else NoNotification
    }
    
    val n1 = getNotification("user@example.com")
    val n2 = getNotification(null)
    
    n1.send("ยินดีต้อนรับ!")
    n2.send("ยินดีต้อนรับ!")  // silent
    println("น2 ส่งแล้ว (แต่ไม่เห็นอะไร)")
}
```

---

## Step 185: Null Safety ในชีวิตจริง

```kotlin
import kotlin.random.Random

// ตัวอย่าง: API Response Handling

data class ApiError(val code: Int, val message: String)

sealed class ApiResponse<out T> {
    data class Success<T>(val data: T) : ApiResponse<T>()
    data class Failure(val error: ApiError) : ApiResponse<Nothing>()
    object NetworkError : ApiResponse<Nothing>()
}

data class UserDto(
    val id: Int?,
    val name: String?,
    val email: String?,
    val role: String?
)

data class UserModel(
    val id: Int,
    val name: String,
    val email: String,
    val role: String
)

// Convert DTO -> Model พร้อม null handling
fun UserDto.toModel(): UserModel? {
    val validId = id ?: return null
    val validName = name?.takeIf { it.isNotBlank() } ?: return null
    val validEmail = email?.takeIf { it.contains("@") } ?: return null
    val validRole = role ?: "user"
    
    return UserModel(validId, validName, validEmail, validRole)
}

// จำลอง API call
fun fetchUser(id: Int): ApiResponse<UserDto> {
    return when (id) {
        1 -> ApiResponse.Success(UserDto(1, "สมชาย", "somchai@email.com", "admin"))
        2 -> ApiResponse.Success(UserDto(2, null, "somying@email.com", null))  // missing name
        3 -> ApiResponse.Success(UserDto(null, "สมศักดิ์", "invalid-email", "user"))  // invalid data
        4 -> ApiResponse.Failure(ApiError(404, "User not found"))
        else -> ApiResponse.NetworkError
    }
}

fun main() {
    println("=== API Response Handling ===")
    
    for (id in 1..5) {
        print("fetchUser($id): ")
        
        when (val response = fetchUser(id)) {
            is ApiResponse.Success -> {
                val dto = response.data
                val model = dto.toModel()
                
                if (model != null) {
                    println("OK - ${model.name} (${model.role})")
                } else {
                    println("ข้อมูลไม่ครบถ้วน - id=${dto.id}, name=${dto.name}, email=${dto.email}")
                }
            }
            is ApiResponse.Failure -> {
                println("Error ${response.error.code}: ${response.error.message}")
            }
            is ApiResponse.NetworkError -> {
                println("Network Error!")
            }
        }
    }
    
    // Chained null safety operations
    println("\n=== Chained Operations ===")
    
    data class Product(val id: Int, val name: String, val price: Double?)
    data class Cart(val products: List<Product>?)
    data class Customer(val name: String, val cart: Cart?)
    
    val customers = listOf(
        Customer("สมชาย", Cart(listOf(
            Product(1, "iPhone", 35000.0),
            Product(2, "AirPods", null)  // ไม่มีราคา
        ))),
        Customer("สมหญิง", Cart(null)),
        Customer("สมศักดิ์", null)
    )
    
    customers.forEach { customer ->
        val total = customer.cart
            ?.products
            ?.mapNotNull { it.price }
            ?.sum()
        
        println("${customer.name}: ${total?.let { "฿$it" } ?: "ไม่มีตะกร้า"}")
    }
    
    // Safe navigation สำหรับ deep nesting
    println("\n=== Deep Nesting ---")
    
    data class Branch(val name: String, val manager: Employee?)
    data class Department(val name: String, val branches: List<Branch>?)
    data class Employee(val name: String, val department: Department?)
    
    val emp1 = Employee("สมชาย", Department("IT", listOf(
        Branch("สาขาหลัก", null),
        Branch("สาขาย่อย", null)  // manager set later
    )))
    
    val branchCount = emp1.department?.branches?.size
    println("${emp1.name} มี $branchCount สาขา")
    
    val firstBranchManager = emp1.department?.branches?.firstOrNull()?.manager?.name
    println("ผู้จัดการสาขาแรก: ${firstBranchManager ?: "ยังไม่มี"}")
}

data class Employee(val name: String, val department: Any? = null)
```

---

## Step 186: Null Safety กับ Collections

```kotlin
fun main() {
    // Collection ที่มี null elements
    val list: List<String?> = listOf("a", null, "b", null, "c")
    
    // filterNotNull - กรอง null ออก
    val noNulls = list.filterNotNull()
    println("filterNotNull: $noNulls")
    
    // mapNotNull - map และกรอง null ออก
    val lengths = list.mapNotNull { it?.length }
    println("mapNotNull lengths: $lengths")
    
    // ต่างกับ map { it?.length }
    val nullableLengths = list.map { it?.length }
    println("map nullable lengths: $nullableLengths")
    
    // Nullable collections
    val nullableList: List<String>? = null
    println("\nnullableList?.size: ${nullableList?.size}")
    println("nullableList.orEmpty(): ${nullableList.orEmpty()}")
    
    val nullableMap: Map<String, Int>? = null
    println("nullableMap.orEmpty(): ${nullableMap.orEmpty()}")
    
    // getOrNull - ปลอดภัยกว่า []
    val numbers = listOf(1, 2, 3, 4, 5)
    println("\nnumbers[10]: ${numbers.getOrNull(10)}")   // null
    println("numbers[2]: ${numbers.getOrNull(2)}")        // 3
    println("getOrElse: ${numbers.getOrElse(10) { -1 }}")  // -1
    
    // Map safe access
    val map = mapOf("a" to 1, "b" to 2)
    println("\nmap[\"c\"]: ${map["c"]}")               // null
    println("getOrDefault: ${map.getOrDefault("c", 0)}")  // 0
    println("getOrElse: ${map.getOrElse("c") { -1 }}")    // -1
    
    // firstOrNull / lastOrNull
    val empty = emptyList<Int>()
    println("\nfirstOrNull: ${empty.firstOrNull()}")  // null แทน exception
    println("lastOrNull: ${empty.lastOrNull()}")
    
    val found = numbers.firstOrNull { it > 3 }
    println("firstOrNull > 3: $found")
    
    val notFound = numbers.firstOrNull { it > 100 }
    println("firstOrNull > 100: $notFound")
    
    // maxOrNull / minOrNull
    println("\nmaxOrNull: ${empty.maxOrNull()}")   // null
    println("minOrNull: ${empty.minOrNull()}")
    println("maxOrNull: ${numbers.maxOrNull()}")   // 5
    
    // randomOrNull
    println("randomOrNull empty: ${empty.randomOrNull()}")  // null
    println("randomOrNull: ${numbers.randomOrNull()}")
    
    // Reduce ปลอดภัย
    println("\nreduceOrNull empty: ${empty.reduceOrNull { a, b -> a + b }}")
    println("reduceOrNull: ${numbers.reduceOrNull { a, b -> a + b }}")
}
```

---

## Step 187: สรุป Null Safety Patterns

```kotlin
// Cheat Sheet: Null Safety ใน Kotlin

fun main() {
    // Pattern 1: ตรวจสอบ null
    val s: String? = "hello"
    
    // a) if check
    if (s != null) println("a) ${s.length}")
    
    // b) ?: Elvis
    println("b) ${s?.length ?: 0}")
    
    // c) let
    s?.let { println("c) ${it.length}") }
    
    // d) !! (ระวัง!)
    println("d) ${s!!.length}")

    // Pattern 2: Default value
    val nullable: String? = null
    val defaulted = nullable ?: "default"
    println("\ndefault: $defaulted")

    // Pattern 3: Transform if non-null
    val upper = nullable?.uppercase()          // null
    val upperNonNull = "hello"?.uppercase()    // HELLO
    println("transform: $upper, $upperNonNull")

    // Pattern 4: Conditional execution
    nullable?.let { doSomething(it) }
    println("conditional: skipped")

    // Pattern 5: throw if null
    val required = nullable ?: throw IllegalStateException("required!")
    // (จะไม่ถึงบรรทัดนี้จริงๆ เพราะ throw ข้างบน)

    // Pattern 6: return if null
    fun processNullable(v: String?): Int {
        val value = v ?: return -1
        return value.length
    }
    println("\nprocessNullable: ${processNullable("hello")}")
    println("processNullable null: ${processNullable(null)}")

    // Pattern 7: takeIf / takeUnless
    val positiveOrNull = 5.takeIf { it > 0 }
    val negativeOrNull = 5.takeIf { it < 0 }
    println("\ntakeIf(>0): $positiveOrNull, takeIf(<0): $negativeOrNull")

    // Pattern 8: safe cast
    val obj: Any = "hello"
    val asString = obj as? String
    val asInt = obj as? Int
    println("\nas String: $asString, as Int: $asInt")

    // Pattern 9: Collection
    val list: List<String?> = listOf("a", null, "b")
    val clean = list.filterNotNull()
    println("\nfilterNotNull: $clean")

    // Pattern 10: Nested null safety
    data class A(val b: B?)
    data class B(val c: C?)
    data class C(val value: String)

    val a = A(B(C("found!")))
    val aNested = A(B(null))
    val aNull = A(null)

    println("\na.b?.c?.value: ${a.b?.c?.value}")
    println("aNested.b?.c?.value: ${aNested.b?.c?.value}")
    println("aNull.b?.c?.value: ${aNull.b?.c?.value}")
}

fun doSomething(s: String) = println("doing: $s")
```

---

## Step 188: ตัวอย่างโปรแกรมจริง - Form Validation

```kotlin
data class RegistrationForm(
    val username: String?,
    val email: String?,
    val password: String?,
    val confirmPassword: String?,
    val age: String?,
    val phone: String?
)

data class ValidationError(val field: String, val message: String)

data class ValidRegistration(
    val username: String,
    val email: String,
    val password: String,
    val age: Int,
    val phone: String?  // optional
)

class FormValidator {
    fun validate(form: RegistrationForm): Pair<ValidRegistration?, List<ValidationError>> {
        val errors = mutableListOf<ValidationError>()
        
        // Validate username
        val username = form.username
            ?.trim()
            ?.takeIf { it.length >= 3 }
            ?: run {
                errors.add(ValidationError("username", 
                    if (form.username.isNullOrBlank()) "username ต้องไม่ว่าง"
                    else "username ต้องมีอย่างน้อย 3 ตัวอักษร"))
                null
            }
        
        // Validate email
        val email = form.email
            ?.trim()
            ?.takeIf { it.matches(Regex("^[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\\.[A-Za-z]{2,}$")) }
            ?: run {
                errors.add(ValidationError("email",
                    if (form.email.isNullOrBlank()) "email ต้องไม่ว่าง"
                    else "email ไม่ถูกต้อง"))
                null
            }
        
        // Validate password
        val password = form.password
            ?.takeIf { it.length >= 8 }
            ?: run {
                errors.add(ValidationError("password",
                    if (form.password.isNullOrBlank()) "password ต้องไม่ว่าง"
                    else "password ต้องมีอย่างน้อย 8 ตัวอักษร"))
                null
            }
        
        // Validate confirm password
        if (password != null && form.confirmPassword != password) {
            errors.add(ValidationError("confirmPassword", "รหัสผ่านไม่ตรงกัน"))
        }
        
        // Validate age
        val age = form.age
            ?.trim()
            ?.toIntOrNull()
            ?.takeIf { it in 13..120 }
            ?: run {
                errors.add(ValidationError("age",
                    when {
                        form.age.isNullOrBlank() -> "อายุต้องไม่ว่าง"
                        form.age.trim().toIntOrNull() == null -> "อายุต้องเป็นตัวเลข"
                        else -> "อายุต้องอยู่ระหว่าง 13-120"
                    }))
                null
            }
        
        // Phone (optional)
        val phone = form.phone
            ?.trim()
            ?.takeIf { it.isNotBlank() }
            ?.takeIf { it.matches(Regex("^[0-9]{9,10}$")) }
            .also { phone ->
                if (form.phone != null && phone == null && form.phone.isNotBlank()) {
                    errors.add(ValidationError("phone", "เบอร์โทรไม่ถูกต้อง (ต้อง 9-10 หลัก)"))
                }
            }
        
        val valid = if (errors.isEmpty() && username != null && email != null && 
                        password != null && age != null) {
            ValidRegistration(username, email, password, age, phone)
        } else null
        
        return Pair(valid, errors)
    }
}

fun main() {
    val validator = FormValidator()
    
    val forms = listOf(
        RegistrationForm("somchai", "somchai@example.com", "Pass123456", "Pass123456", "25", "0812345678"),
        RegistrationForm(null, "invalid", "short", "different", "abc", "12345"),
        RegistrationForm("ab", "valid@email.com", "Password123", "Password123", "15", null),
        RegistrationForm("validUser", "user@test.com", "SecurePass1", "SecurePass1", "30", "")
    )
    
    forms.forEachIndexed { index, form ->
        println("\n=== Form ${index + 1} ===")
        val (valid, errors) = validator.validate(form)
        
        if (valid != null) {
            println("สำเร็จ! ลงทะเบียน: ${valid.username} (${valid.email})")
        } else {
            println("ข้อผิดพลาด:")
            errors.forEach { error ->
                println("  ${error.field}: ${error.message}")
            }
        }
    }
}
```

---

## Step 189: Null Safety กับ Coroutines Preview

```kotlin
// Preview: Null Safety ใน async code (Coroutines จะเรียนในส่วนต่อไป)

// จำลอง async operations ด้วย nullable results
class UserService {
    private val db = mapOf(
        1 to mapOf("name" to "สมชาย", "email" to "somchai@email.com"),
        2 to mapOf("name" to "สมหญิง", "email" to null)
    )
    
    fun getUser(id: Int): Map<String, String?>? = db[id]
    
    fun getUserName(id: Int): String? = getUser(id)?.get("name")
    
    fun getUserEmail(id: Int): String? = getUser(id)?.get("email")
}

class OrderService {
    private val orders = mapOf(
        1 to listOf(mapOf("item" to "iPhone", "price" to "35000")),
        2 to emptyList<Map<String, String>>()
    )
    
    fun getOrders(userId: Int): List<Map<String, String>>? = orders[userId]
    
    fun getOrderTotal(userId: Int): Double? {
        return orders[userId]
            ?.takeIf { it.isNotEmpty() }
            ?.sumOf { it["price"]?.toDoubleOrNull() ?: 0.0 }
    }
}

fun main() {
    val userService = UserService()
    val orderService = OrderService()
    
    println("=== User Dashboard ===")
    
    for (userId in 1..3) {
        println("\nUser ID: $userId")
        
        val name = userService.getUserName(userId) ?: "ไม่พบผู้ใช้"
        println("ชื่อ: $name")
        
        if (userService.getUser(userId) != null) {
            val email = userService.getUserEmail(userId) ?: "ไม่มี email"
            println("Email: $email")
            
            val total = orderService.getOrderTotal(userId)
            println("ยอดสั่งซื้อ: ${total?.let { "฿$it" } ?: "ยังไม่มีการสั่งซื้อ"}")
        }
    }
    
    // Null safety chain
    println("\n=== Chained Operations ===")
    
    data class Config(val settings: Map<String, String>?)
    
    val config = Config(mapOf("timeout" to "30", "retry" to "3"))
    val emptyConfig = Config(null)
    
    val timeout = config.settings?.get("timeout")?.toIntOrNull() ?: 30
    val retry = config.settings?.get("retry")?.toIntOrNull() ?: 3
    val maxConn = config.settings?.get("maxConn")?.toIntOrNull() ?: 10
    
    println("timeout: $timeout, retry: $retry, maxConn: $maxConn")
    
    val emptyTimeout = emptyConfig.settings?.get("timeout")?.toIntOrNull() ?: 30
    println("emptyConfig timeout: $emptyTimeout")
    
    // Null coalescing chain
    println("\n=== Multiple fallbacks ===")
    
    val primary: String? = null
    val secondary: String? = null
    val tertiary: String? = "tertiary value"
    val final: String = "final fallback"
    
    val result = primary ?: secondary ?: tertiary ?: final
    println("result: $result")
}
```

---

## Step 190: แบบฝึกหัด Null Safety

```kotlin
// ============================================================
// Exercise 1: Safe Config Parser
// ============================================================

class ConfigParser {
    fun parse(input: String?): Map<String, String> {
        val result = mutableMapOf<String, String>()
        
        input?.lines()
            ?.filter { it.isNotBlank() && !it.startsWith("#") }
            ?.forEach { line ->
                val parts = line.split("=", limit = 2)
                if (parts.size == 2) {
                    val key = parts[0].trim().takeIf { it.isNotEmpty() }
                    val value = parts[1].trim()
                    key?.let { result[it] = value }
                }
            }
        
        return result
    }
    
    fun getInt(config: Map<String, String>, key: String, default: Int = 0): Int {
        return config[key]?.toIntOrNull() ?: default
    }
    
    fun getString(config: Map<String, String>, key: String, default: String = ""): String {
        return config[key]?.takeIf { it.isNotBlank() } ?: default
    }
    
    fun getBoolean(config: Map<String, String>, key: String, default: Boolean = false): Boolean {
        return when (config[key]?.lowercase()) {
            "true", "yes", "1" -> true
            "false", "no", "0" -> false
            else -> default
        }
    }
}

fun main() {
    val configText = """
        # Database config
        db.host = localhost
        db.port = 5432
        db.name = mydb
        db.ssl = true
        
        # App config
        app.name = MyApp
        app.port = 8080
        app.debug = false
        
        # Empty value
        app.secret = 
    """.trimIndent()
    
    val parser = ConfigParser()
    val config = parser.parse(configText)
    
    println("=== Config ===")
    config.forEach { (k, v) -> println("  $k = $v") }
    
    println("\n=== Parsed Values ===")
    println("db.host: ${parser.getString(config, "db.host")}")
    println("db.port: ${parser.getInt(config, "db.port")}")
    println("db.ssl: ${parser.getBoolean(config, "db.ssl")}")
    println("app.name: ${parser.getString(config, "app.name")}")
    println("app.debug: ${parser.getBoolean(config, "app.debug")}")
    println("app.secret: '${parser.getString(config, "app.secret", "default-secret")}'")
    println("missing: ${parser.getString(config, "missing", "not found")}")
    
    // Null config
    val emptyConfig = parser.parse(null)
    println("\nEmpty config port: ${parser.getInt(emptyConfig, "db.port", 3306)}")
    
    // ============================================================
    // Exercise 2: Repository Pattern กับ Null Safety
    // ============================================================
    
    data class Product(val id: Int, val name: String, val price: Double, val stock: Int?)
    
    interface ProductRepository {
        fun findById(id: Int): Product?
        fun findByName(name: String): List<Product>
        fun save(product: Product): Product
        fun updateStock(id: Int, stock: Int): Product?
    }
    
    class InMemoryProductRepository : ProductRepository {
        private val products = mutableMapOf<Int, Product>()
        private var nextId = 1
        
        override fun findById(id: Int) = products[id]
        
        override fun findByName(name: String) = products.values
            .filter { it.name.contains(name, ignoreCase = true) }
        
        override fun save(product: Product): Product {
            val saved = if (product.id == 0) product.copy(id = nextId++) else product
            products[saved.id] = saved
            return saved
        }
        
        override fun updateStock(id: Int, stock: Int): Product? {
            val product = products[id] ?: return null
            val updated = product.copy(stock = stock)
            products[id] = updated
            return updated
        }
    }
    
    println("\n=== Product Repository ===")
    val repo = InMemoryProductRepository()
    
    repo.save(Product(0, "iPhone 15", 35000.0, 10))
    repo.save(Product(0, "Samsung S24", 28000.0, null))  // stock unknown
    repo.save(Product(0, "MacBook Pro", 65000.0, 5))
    
    val product = repo.findById(1)
    println("findById(1): $product")
    println("findById(99): ${repo.findById(99) ?: "ไม่พบ"}")
    
    val stockInfo = product?.stock
    println("stock: ${stockInfo?.let { "มี $it ชิ้น" } ?: "ไม่ทราบจำนวน"}")
    
    val updated = repo.updateStock(2, 15)
    println("updated stock: ${updated?.stock}")
    
    val notFound = repo.updateStock(99, 5)
    println("update not found: $notFound")
    
    // findByName
    val phones = repo.findByName("phone")
    println("\nphones: ${phones.map { it.name }}")
    
    // สรุป stock
    val allProducts = listOf(1, 2, 3).mapNotNull { repo.findById(it) }
    val knownStock = allProducts.mapNotNull { it.stock }
    val unknownCount = allProducts.count { it.stock == null }
    println("\nสินค้าทั้งหมด: ${allProducts.size}")
    println("ทราบจำนวน: ${knownStock.sum()} ชิ้น")
    println("ไม่ทราบจำนวน: $unknownCount รายการ")
}
```

---

## สรุปส่วนที่ 9 (Summary)

| Operator | ความหมาย | ตัวอย่าง |
|---------|----------|---------|
| `Type?` | Nullable type | `String?` |
| `?.` | Safe call | `str?.length` |
| `?:` | Elvis operator | `str ?: "default"` |
| `!!` | Non-null assertion | `str!!.length` |
| `as?` | Safe cast | `obj as? String` |
| `is` | Type check (smart cast) | `if (obj is String)` |

### ลำดับการใช้งาน (แนะนำ)
1. ใช้ `?.` สำหรับ method calls บน nullable
2. ใช้ `?:` สำหรับ default value
3. ใช้ `let` สำหรับ conditional execution
4. ใช้ `requireNotNull/checkNotNull` สำหรับ fail-fast
5. ใช้ `!!` เฉพาะเมื่อแน่ใจ 100% ว่าไม่ null

---

[← Part 08: Interface](../part08/README.md) | [Part 10: Lambda →](../part10/README.md)
