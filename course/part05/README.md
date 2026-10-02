# Part 05: Collections - List, Set, Map

## ภาพรวม (Overview)
ในส่วนนี้เราจะเรียนรู้เกี่ยวกับ Collections ใน Kotlin ซึ่งเป็นโครงสร้างข้อมูลที่ใช้เก็บกลุ่มของข้อมูล
ประกอบด้วย List, Set, Map และ operators ต่างๆ ที่ใช้จัดการข้อมูล

**Steps ที่ครอบคลุม:** 71-90

---

## Step 71: Introduction to Collections

```kotlin
// Collections ใน Kotlin แบ่งเป็น 2 ประเภทหลัก:
// 1. Immutable (Read-only) - ไม่สามารถเปลี่ยนแปลงได้หลังสร้าง
// 2. Mutable - สามารถเพิ่ม/ลบ/แก้ไขได้

fun main() {
    // Immutable List - ไม่สามารถแก้ไขได้
    val immutableList = listOf(1, 2, 3, 4, 5)
    println("Immutable List: $immutableList")
    // immutableList.add(6) // Error! ทำไม่ได้

    // Mutable List - แก้ไขได้
    val mutableList = mutableListOf(1, 2, 3, 4, 5)
    mutableList.add(6)
    println("Mutable List: $mutableList")

    // Immutable Set - ไม่มีข้อมูลซ้ำ
    val immutableSet = setOf(1, 2, 3, 2, 1)
    println("Immutable Set: $immutableSet") // {1, 2, 3}

    // Immutable Map - คู่ key-value
    val immutableMap = mapOf("name" to "สมชาย", "age" to 25)
    println("Immutable Map: $immutableMap")

    // ตรวจสอบประเภทของ Collections
    println("Type of immutableList: ${immutableList::class.simpleName}")
    println("Type of mutableList: ${mutableList::class.simpleName}")
}
```

---

## Step 72: List Creation - การสร้าง List

```kotlin
fun main() {
    // วิธีสร้าง List แบบต่างๆ

    // 1. listOf() - สร้าง immutable list
    val fruits = listOf("แอปเปิ้ล", "กล้วย", "ส้ม", "มะม่วง")
    println("fruits: $fruits")

    // 2. mutableListOf() - สร้าง mutable list
    val vegetables = mutableListOf("ผักบุ้ง", "กะหล่ำปลี")
    vegetables.add("แครอท")
    vegetables.add(0, "มะเขือเทศ") // เพิ่มที่ตำแหน่ง 0
    println("vegetables: $vegetables")

    // 3. ArrayList() - Java-style
    val numbers = ArrayList<Int>()
    numbers.add(10)
    numbers.add(20)
    numbers.add(30)
    println("numbers: $numbers")

    // 4. List() constructor - สร้างด้วย lambda
    val squares = List(5) { index -> index * index }
    println("squares: $squares") // [0, 1, 4, 9, 16]

    // 5. emptyList() - list ว่าง
    val emptyList = emptyList<String>()
    println("emptyList: $emptyList")
    println("isEmpty: ${emptyList.isEmpty()}")

    // 6. buildList - สร้าง list แบบ DSL
    val builtList = buildList {
        add("ข้าว")
        add("ก๋วยเตี๋ยว")
        addAll(listOf("ส้มตำ", "ต้มยำ"))
    }
    println("builtList: $builtList")

    // การเข้าถึงข้อมูลใน List
    println("\n--- การเข้าถึงข้อมูล ---")
    println("ผลไม้ตัวแรก: ${fruits[0]}")
    println("ผลไม้ตัวสุดท้าย: ${fruits[fruits.size - 1]}")
    println("ผลไม้ตัวสุดท้าย (first): ${fruits.first()}")
    println("ผลไม้ตัวสุดท้าย (last): ${fruits.last()}")
    println("ขนาด list: ${fruits.size}")

    // การ iterate
    println("\n--- การ iterate ---")
    for (fruit in fruits) {
        print("$fruit ")
    }
    println()

    fruits.forEachIndexed { index, fruit ->
        println("[$index] $fruit")
    }
}
```

---

## Step 73: List Operations - การดำเนินการกับ List

```kotlin
fun main() {
    val numbers = mutableListOf(5, 2, 8, 1, 9, 3, 7, 4, 6)

    println("Original: $numbers")

    // การเรียงลำดับ
    val sorted = numbers.sorted()
    println("sorted (asc): $sorted")

    val sortedDesc = numbers.sortedDescending()
    println("sorted (desc): $sortedDesc")

    // เรียงลำดับ in-place (เปลี่ยนแปลง list ต้นฉบับ)
    numbers.sort()
    println("After sort(): $numbers")

    // การค้นหา
    val fruits = listOf("แอปเปิ้ล", "กล้วย", "ส้ม", "มะม่วง", "กล้วย")
    println("\n--- การค้นหา ---")
    println("contains กล้วย: ${fruits.contains("กล้วย")}")
    println("indexOf กล้วย: ${fruits.indexOf("กล้วย")}")
    println("lastIndexOf กล้วย: ${fruits.lastIndexOf("กล้วย")}")

    // การตัด subList
    val subList = fruits.subList(1, 3)
    println("subList(1,3): $subList")

    // การรวม Lists
    val list1 = listOf(1, 2, 3)
    val list2 = listOf(4, 5, 6)
    val combined = list1 + list2
    println("\ncombined: $combined")

    // การลบออกจาก list (สร้าง list ใหม่)
    val removed = list1 - 2
    println("removed 2: $removed")

    // MutableList operations
    val mList = mutableListOf(1, 2, 3, 4, 5)
    mList.removeAt(0)        // ลบที่ index 0
    mList.remove(3)          // ลบค่า 3
    println("\nAfter removes: $mList")

    mList.set(0, 99)         // เปลี่ยนค่าที่ index 0
    println("After set(0, 99): $mList")

    mList.addAll(listOf(10, 20))  // เพิ่มหลายค่า
    println("After addAll: $mList")

    mList.clear()
    println("After clear: $mList")

    // การแปลง
    val strNumbers = listOf("1", "2", "3", "4", "5")
    val intNumbers = strNumbers.map { it.toInt() }
    println("\nConverted to Int: $intNumbers")

    // toList() และ toMutableList()
    val immutable = listOf(1, 2, 3)
    val mutable = immutable.toMutableList()
    mutable.add(4)
    println("mutable from immutable: $mutable")
}
```

---

## Step 74: Set - ชุดข้อมูลไม่ซ้ำ

```kotlin
fun main() {
    // Set - เก็บข้อมูลโดยไม่มีการซ้ำ และไม่มีลำดับ (ส่วนใหญ่)

    // 1. setOf() - immutable set
    val colors = setOf("แดง", "เขียว", "น้ำเงิน", "แดง", "เขียว")
    println("colors: $colors") // ไม่มีซ้ำ: {แดง, เขียว, น้ำเงิน}
    println("size: ${colors.size}") // 3

    // 2. mutableSetOf() - mutable set
    val animals = mutableSetOf("สุนัข", "แมว", "กระต่าย")
    animals.add("นก")
    animals.add("สุนัข") // ไม่เพิ่มเพราะมีอยู่แล้ว
    println("\nanimals: $animals")

    // 3. HashSet - ลำดับไม่แน่นอน
    val hashSet = hashSetOf(3, 1, 4, 1, 5, 9, 2, 6)
    println("\nhashSet: $hashSet")

    // 4. LinkedHashSet - รักษาลำดับการเพิ่ม
    val linkedSet = linkedSetOf(3, 1, 4, 1, 5, 9, 2, 6)
    println("linkedSet: $linkedSet")

    // 5. TreeSet - เรียงลำดับอัตโนมัติ
    val sortedSet = sortedSetOf(3, 1, 4, 1, 5, 9, 2, 6)
    println("sortedSet: $sortedSet")

    // การดำเนินการกับ Set
    val set1 = setOf(1, 2, 3, 4, 5)
    val set2 = setOf(4, 5, 6, 7, 8)

    println("\n--- Set Operations ---")
    println("set1: $set1")
    println("set2: $set2")

    // Union (รวมกัน)
    println("union: ${set1.union(set2)}")
    println("plus (+): ${set1 + set2}")

    // Intersect (ส่วนที่ตัดกัน)
    println("intersect: ${set1.intersect(set2)}")

    // Subtract (ส่วนต่าง)
    println("subtract: ${set1.subtract(set2)}")
    println("minus (-): ${set1 - set2}")

    // ตรวจสอบ
    println("\ncontains 3: ${set1.contains(3)}")
    println("3 in set1: ${3 in set1}")
    println("containsAll: ${set1.containsAll(setOf(1, 2))}")

    // แปลง Set เป็น List
    val listFromSet = colors.toList()
    println("\nSet to List: $listFromSet")

    // แปลง List เป็น Set (ลบซ้ำ)
    val listWithDuplicates = listOf(1, 2, 3, 2, 1, 4, 3)
    val setFromList = listWithDuplicates.toSet()
    println("List to Set: $setFromList")
    println("Distinct: ${listWithDuplicates.distinct()}")
}
```

---

## Step 75: Map - คู่ Key-Value

```kotlin
fun main() {
    // Map - เก็บข้อมูลเป็นคู่ key-value

    // 1. mapOf() - immutable map
    val studentGrades = mapOf(
        "สมชาย" to 85,
        "สมหญิง" to 92,
        "สมศักดิ์" to 78,
        "สมใจ" to 95
    )
    println("studentGrades: $studentGrades")

    // 2. mutableMapOf() - mutable map
    val inventory = mutableMapOf(
        "แอปเปิ้ล" to 50,
        "กล้วย" to 30,
        "ส้ม" to 45
    )
    inventory["มะม่วง"] = 25           // เพิ่มหรือแก้ไข
    inventory["แอปเปิ้ล"] = 60         // แก้ไขค่าที่มีอยู่
    inventory.remove("กล้วย")           // ลบ
    println("\ninventory: $inventory")

    // 3. HashMap - ลำดับไม่แน่นอน
    val hashMap = hashMapOf("a" to 1, "b" to 2, "c" to 3)

    // 4. LinkedHashMap - รักษาลำดับการเพิ่ม
    val linkedMap = linkedMapOf("a" to 1, "b" to 2, "c" to 3)

    // 5. TreeMap - เรียงตาม key
    val sortedMap = sortedMapOf("c" to 3, "a" to 1, "b" to 2)
    println("\nsortedMap: $sortedMap")

    // การเข้าถึงข้อมูล
    println("\n--- การเข้าถึงข้อมูล ---")
    println("สมชาย: ${studentGrades["สมชาย"]}")
    println("สมชาย (getOrDefault): ${studentGrades.getOrDefault("สมชาย", 0)}")
    println("ไม่มีชื่อนี้: ${studentGrades["สมพร"]}")  // null
    println("ไม่มีชื่อนี้ (getOrDefault): ${studentGrades.getOrDefault("สมพร", 0)}")

    // iterate
    println("\n--- iterate ---")
    for ((name, grade) in studentGrades) {
        println("$name: $grade")
    }

    studentGrades.forEach { (name, grade) ->
        println("นักเรียน: $name, คะแนน: $grade")
    }

    // keys, values, entries
    println("\nkeys: ${studentGrades.keys}")
    println("values: ${studentGrades.values}")
    println("entries: ${studentGrades.entries}")

    // ตรวจสอบ
    println("\ncontainsKey สมชาย: ${studentGrades.containsKey("สมชาย")}")
    println("containsValue 92: ${studentGrades.containsValue(92)}")
    println("isEmpty: ${studentGrades.isEmpty()}")
    println("size: ${studentGrades.size}")

    // buildMap
    val builtMap = buildMap<String, Int> {
        put("ก", 1)
        put("ข", 2)
        putAll(mapOf("ค" to 3, "ง" to 4))
    }
    println("\nbuiltMap: $builtMap")
}
```

---

## Step 76: Collection Operators - filter

```kotlin
fun main() {
    val numbers = listOf(1, 2, 3, 4, 5, 6, 7, 8, 9, 10)
    val students = listOf(
        mapOf("name" to "สมชาย", "grade" to 85, "age" to 20),
        mapOf("name" to "สมหญิง", "grade" to 92, "age" to 22),
        mapOf("name" to "สมศักดิ์", "grade" to 78, "age" to 21),
        mapOf("name" to "สมใจ", "grade" to 95, "age" to 19),
        mapOf("name" to "สมพร", "grade" to 65, "age" to 23)
    )

    // filter - กรองข้อมูลตามเงื่อนไข
    println("--- filter ---")
    val evenNumbers = numbers.filter { it % 2 == 0 }
    println("เลขคู่: $evenNumbers")

    val bigNumbers = numbers.filter { it > 5 }
    println("มากกว่า 5: $bigNumbers")

    // filterNot - กรองข้อมูลที่ไม่ตรงเงื่อนไข
    val oddNumbers = numbers.filterNot { it % 2 == 0 }
    println("เลขคี่: $oddNumbers")

    // filterIndexed - กรองพร้อม index
    val evenIndexed = numbers.filterIndexed { index, _ -> index % 2 == 0 }
    println("ตำแหน่งคู่: $evenIndexed")

    // filterNotNull - กรอง null ออก
    val withNulls = listOf(1, null, 3, null, 5)
    val noNulls = withNulls.filterNotNull()
    println("ไม่มี null: $noNulls")

    // filterIsInstance - กรองตามประเภท
    val mixed: List<Any> = listOf(1, "hello", 2.5, "world", 3)
    val strings = mixed.filterIsInstance<String>()
    println("เฉพาะ String: $strings")

    // กรองนักเรียนที่ผ่าน (>= 80)
    val passedStudents = students.filter { it["grade"] as Int >= 80 }
    println("\nนักเรียนที่ผ่าน:")
    passedStudents.forEach { println("  ${it["name"]}: ${it["grade"]}") }

    // กรองหลายเงื่อนไข
    val youngHighAchievers = students.filter { 
        (it["grade"] as Int) >= 90 && (it["age"] as Int) <= 21
    }
    println("\nเยาวชนคะแนนสูง:")
    youngHighAchievers.forEach { println("  ${it["name"]}: grade=${it["grade"]}, age=${it["age"]}") }

    // partition - แบ่งเป็น 2 กลุ่ม
    val (passed, failed) = numbers.partition { it > 5 }
    println("\nมากกว่า 5: $passed")
    println("ไม่เกิน 5: $failed")
}
```

---

## Step 77: Collection Operators - map และ transform

```kotlin
data class Product(val name: String, val price: Double, val category: String)

fun main() {
    val products = listOf(
        Product("iPhone", 35000.0, "มือถือ"),
        Product("Samsung", 28000.0, "มือถือ"),
        Product("MacBook", 65000.0, "คอมพิวเตอร์"),
        Product("Dell", 45000.0, "คอมพิวเตอร์"),
        Product("AirPods", 8000.0, "อุปกรณ์เสริม")
    )

    // map - แปลงข้อมูลแต่ละตัว
    println("--- map ---")
    val names = products.map { it.name }
    println("ชื่อสินค้า: $names")

    val prices = products.map { it.price }
    println("ราคา: $prices")

    val pricesWithVat = products.map { it.price * 1.07 }
    println("ราคารวม VAT: $pricesWithVat")

    // mapIndexed - map พร้อม index
    val numberedProducts = products.mapIndexed { index, product ->
        "${index + 1}. ${product.name}"
    }
    println("\nรายการสินค้า: $numberedProducts")

    // mapNotNull - map และกรอง null
    val numbers = listOf("1", "abc", "3", "xyz", "5")
    val validNumbers = numbers.mapNotNull { it.toIntOrNull() }
    println("\nเลขที่ valid: $validNumbers")

    // flatMap - แปลงและ flatten
    println("\n--- flatMap ---")
    val sentences = listOf("Hello World", "Kotlin is fun", "Collections are powerful")
    val words = sentences.flatMap { it.split(" ") }
    println("คำทั้งหมด: $words")

    val matrix = listOf(listOf(1, 2, 3), listOf(4, 5, 6), listOf(7, 8, 9))
    val flat = matrix.flatten()
    println("flatten matrix: $flat")

    // zip - จับคู่ 2 lists
    println("\n--- zip ---")
    val names2 = listOf("อ", "ข", "ค")
    val values = listOf(1, 2, 3)
    val zipped = names2.zip(values)
    println("zipped: $zipped")

    val (unzippedNames, unzippedValues) = zipped.unzip()
    println("unzipped names: $unzippedNames")
    println("unzipped values: $unzippedValues")

    // zipWithNext - จับคู่ element ที่ติดกัน
    val sequence = listOf(1, 3, 6, 10, 15)
    val differences = sequence.zipWithNext { a, b -> b - a }
    println("ผลต่าง: $differences")

    // associate - แปลงเป็น Map
    println("\n--- associate ---")
    val productMap = products.associate { it.name to it.price }
    println("product map: $productMap")

    val productByName = products.associateBy { it.name }
    println("by name: ${productByName.keys}")
}
```

---

## Step 78: reduce, fold, และ aggregate

```kotlin
fun main() {
    val numbers = listOf(1, 2, 3, 4, 5, 6, 7, 8, 9, 10)
    val prices = listOf(100.0, 250.5, 75.0, 320.0, 150.0)

    // reduce - รวมข้อมูลโดยไม่มีค่าเริ่มต้น
    println("--- reduce ---")
    val sum = numbers.reduce { acc, num -> acc + num }
    println("ผลรวม: $sum")

    val product = numbers.take(5).reduce { acc, num -> acc * num }
    println("ผลคูณ 5 ตัวแรก: $product")

    val maxByReduce = numbers.reduce { acc, num -> if (num > acc) num else acc }
    println("ค่าสูงสุด: $maxByReduce")

    // reduceOrNull - ไม่ throw exception เมื่อ list ว่าง
    val empty = emptyList<Int>()
    val result = empty.reduceOrNull { acc, num -> acc + num }
    println("ผลลัพธ์จาก empty list: $result") // null

    // fold - รวมข้อมูลโดยมีค่าเริ่มต้น
    println("\n--- fold ---")
    val sumWithFold = numbers.fold(0) { acc, num -> acc + num }
    println("ผลรวมด้วย fold: $sumWithFold")

    // fold เริ่มด้วย 100
    val sumFrom100 = numbers.fold(100) { acc, num -> acc + num }
    println("ผลรวมจาก 100: $sumFrom100")

    // fold สร้าง string
    val sentence = listOf("Hello", "World", "from", "Kotlin")
    val joined = sentence.fold("") { acc, word ->
        if (acc.isEmpty()) word else "$acc $word"
    }
    println("joined: $joined")

    // foldRight - fold จากขวาไปซ้าย
    val reversedJoin = sentence.foldRight("") { word, acc ->
        if (acc.isEmpty()) word else "$word $acc"
    }
    println("foldRight: $reversedJoin")

    // aggregate functions
    println("\n--- aggregate ---")
    println("sum: ${numbers.sum()}")
    println("count: ${numbers.count()}")
    println("average: ${numbers.average()}")
    println("max: ${numbers.max()}")
    println("min: ${numbers.min()}")
    println("maxOrNull: ${numbers.maxOrNull()}")
    println("minOrNull: ${numbers.minOrNull()}")

    // sumOf, maxOf, minOf
    data class Student(val name: String, val grade: Int)
    val students = listOf(
        Student("สมชาย", 85),
        Student("สมหญิง", 92),
        Student("สมศักดิ์", 78)
    )
    println("\nผลรวมคะแนน: ${students.sumOf { it.grade }}")
    println("คะแนนสูงสุด: ${students.maxOf { it.grade }}")
    println("คะแนนต่ำสุด: ${students.minOf { it.grade }}")

    // running operations
    println("\n--- running operations ---")
    val runningSum = numbers.runningFold(0) { acc, num -> acc + num }
    println("running sum: $runningSum")

    val runningReduce = numbers.runningReduce { acc, num -> acc + num }
    println("running reduce: $runningReduce")
}
```

---

## Step 79: groupBy และ sortedBy

```kotlin
data class Employee(
    val name: String,
    val department: String,
    val salary: Int,
    val level: String
)

fun main() {
    val employees = listOf(
        Employee("สมชาย", "IT", 50000, "Senior"),
        Employee("สมหญิง", "HR", 45000, "Junior"),
        Employee("สมศักดิ์", "IT", 60000, "Lead"),
        Employee("สมใจ", "Finance", 55000, "Senior"),
        Employee("สมพร", "HR", 42000, "Junior"),
        Employee("สมบัติ", "IT", 48000, "Junior"),
        Employee("สมคิด", "Finance", 65000, "Lead"),
        Employee("สมนึก", "HR", 58000, "Senior")
    )

    // groupBy - จัดกลุ่มข้อมูล
    println("--- groupBy ---")
    val byDepartment = employees.groupBy { it.department }
    for ((dept, emps) in byDepartment) {
        println("$dept: ${emps.map { it.name }}")
    }

    val byLevel = employees.groupBy { it.level }
    println("\nกลุ่มตาม level:")
    for ((level, emps) in byLevel) {
        println("$level: ${emps.size} คน")
    }

    // groupBy พร้อมแปลงค่า
    val deptToSalaries = employees.groupBy(
        keySelector = { it.department },
        valueTransform = { it.salary }
    )
    println("\nเงินเดือนตาม department:")
    deptToSalaries.forEach { (dept, salaries) ->
        println("$dept: avg=${salaries.average().toInt()}, total=${salaries.sum()}")
    }

    // sortedBy - เรียงลำดับ
    println("\n--- sortedBy ---")
    val sortedBySalary = employees.sortedBy { it.salary }
    sortedBySalary.forEach { println("${it.name}: ${it.salary}") }

    val sortedBySalaryDesc = employees.sortedByDescending { it.salary }
    println("\nเรียงมากไปน้อย:")
    sortedBySalaryDesc.take(3).forEach { println("${it.name}: ${it.salary}") }

    // sortedWith - เรียงหลายเงื่อนไข
    val sortedMultiple = employees.sortedWith(
        compareBy({ it.department }, { it.salary })
    )
    println("\nเรียงตาม dept แล้ว salary:")
    sortedMultiple.forEach { println("${it.department} - ${it.name}: ${it.salary}") }

    // minByOrNull, maxByOrNull
    val highestPaid = employees.maxByOrNull { it.salary }
    val lowestPaid = employees.minByOrNull { it.salary }
    println("\nเงินเดือนสูงสุด: ${highestPaid?.name} = ${highestPaid?.salary}")
    println("เงินเดือนต่ำสุด: ${lowestPaid?.name} = ${lowestPaid?.salary}")

    // chunked - แบ่ง list เป็นกลุ่มๆ
    println("\n--- chunked ---")
    val chunks = employees.chunked(3)
    chunks.forEachIndexed { index, chunk ->
        println("กลุ่ม ${index + 1}: ${chunk.map { it.name }}")
    }

    // windowed - sliding window
    val numbers = listOf(1, 2, 3, 4, 5, 6, 7)
    val windows = numbers.windowed(3)
    println("\nwindowed(3): $windows")

    val windowedStep2 = numbers.windowed(3, step = 2)
    println("windowed(3, step=2): $windowedStep2")
}
```

---

## Step 80: any, all, none, count

```kotlin
data class Order(val id: Int, val product: String, val amount: Int, val status: String)

fun main() {
    val orders = listOf(
        Order(1, "iPhone", 2, "completed"),
        Order(2, "MacBook", 1, "pending"),
        Order(3, "AirPods", 5, "completed"),
        Order(4, "iPad", 1, "cancelled"),
        Order(5, "Apple Watch", 3, "pending"),
        Order(6, "iMac", 1, "completed")
    )

    // any - มีอย่างน้อย 1 ตัวที่ตรงเงื่อนไข
    println("--- any ---")
    val hasCompleted = orders.any { it.status == "completed" }
    println("มี order ที่ completed: $hasCompleted")

    val hasBigOrder = orders.any { it.amount > 4 }
    println("มี order มากกว่า 4: $hasBigOrder")

    val hasDelivered = orders.any { it.status == "delivered" }
    println("มี order ที่ delivered: $hasDelivered")

    // all - ทุกตัวตรงเงื่อนไข
    println("\n--- all ---")
    val allCompleted = orders.all { it.status == "completed" }
    println("ทุก order completed: $allCompleted")

    val allPositiveAmount = orders.all { it.amount > 0 }
    println("ทุก order มีจำนวน > 0: $allPositiveAmount")

    // none - ไม่มีตัวใดตรงเงื่อนไข
    println("\n--- none ---")
    val noneDelivered = orders.none { it.status == "delivered" }
    println("ไม่มี order ที่ delivered: $noneDelivered")

    val noneNegative = orders.none { it.amount < 0 }
    println("ไม่มีจำนวนติดลบ: $noneNegative")

    // count - นับจำนวนที่ตรงเงื่อนไข
    println("\n--- count ---")
    println("จำนวนทั้งหมด: ${orders.count()}")
    println("จำนวน completed: ${orders.count { it.status == "completed" }}")
    println("จำนวน pending: ${orders.count { it.status == "pending" }}")
    println("จำนวนที่มี amount > 2: ${orders.count { it.amount > 2 }}")

    // find - หาตัวแรกที่ตรงเงื่อนไข
    println("\n--- find ---")
    val firstPending = orders.find { it.status == "pending" }
    println("pending ตัวแรก: $firstPending")

    val firstBigOrder = orders.findLast { it.amount >= 3 }
    println("big order ตัวสุดท้าย: $firstBigOrder")

    // indexOf ด้วย predicate
    val firstCompletedIndex = orders.indexOfFirst { it.status == "completed" }
    val lastCompletedIndex = orders.indexOfLast { it.status == "completed" }
    println("\nindex ของ completed แรก: $firstCompletedIndex")
    println("index ของ completed สุดท้าย: $lastCompletedIndex")

    // take / drop
    println("\n--- take/drop ---")
    println("take(3): ${orders.take(3).map { it.id }}")
    println("drop(3): ${orders.drop(3).map { it.id }}")
    println("takeLast(2): ${orders.takeLast(2).map { it.id }}")
    println("dropLast(2): ${orders.dropLast(2).map { it.id }}")

    // takeWhile / dropWhile
    val amounts = listOf(1, 2, 3, 5, 8, 13, 21)
    println("\ntakeWhile < 10: ${amounts.takeWhile { it < 10 }}")
    println("dropWhile < 10: ${amounts.dropWhile { it < 10 }}")
}
```

---

## Step 81: Sequences - การประมวลผลแบบ Lazy

```kotlin
fun main() {
    // Sequence - ประมวลผลแบบ lazy (ทีละ element)
    // เหมาะกับ collection ขนาดใหญ่หรือ operation หลายขั้นตอน

    println("--- Sequence vs List ---")

    // List: ประมวลผลทั้ง collection ในแต่ละ step
    val listResult = (1..10)
        .filter { 
            print("filter($it) ")
            it % 2 == 0 
        }
        .map { 
            print("map($it) ")
            it * it 
        }
        .first()
    println("\nList result: $listResult")

    println()

    // Sequence: ประมวลผลทีละ element จนเสร็จ
    val seqResult = (1..10).asSequence()
        .filter { 
            print("filter($it) ")
            it % 2 == 0 
        }
        .map { 
            print("map($it) ")
            it * it 
        }
        .first()
    println("\nSequence result: $seqResult")

    // สร้าง Sequence แบบต่างๆ
    println("\n--- สร้าง Sequence ---")

    // sequenceOf
    val seq1 = sequenceOf(1, 2, 3, 4, 5)
    println("sequenceOf: ${seq1.toList()}")

    // asSequence จาก collection
    val seq2 = listOf(1, 2, 3).asSequence()
    println("asSequence: ${seq2.toList()}")

    // generateSequence - สร้าง sequence แบบ infinite
    val naturals = generateSequence(1) { it + 1 }
    val first10 = naturals.take(10).toList()
    println("first 10 naturals: $first10")

    // Fibonacci sequence
    val fibonacci = generateSequence(Pair(0, 1)) { (a, b) -> Pair(b, a + b) }
        .map { it.first }
        .take(10)
        .toList()
    println("Fibonacci: $fibonacci")

    // sequence builder
    val evenSquares = sequence {
        var n = 0
        while (true) {
            yield(n * n)
            n += 2
        }
    }
    println("even squares: ${evenSquares.take(8).toList()}")

    // ประสิทธิภาพ: Sequence ดีกว่า List เมื่อมีหลาย operations
    println("\n--- Performance Example ---")
    val largeRange = (1..1_000_000)

    // ด้วย List: สร้าง intermediate list
    val listTime = measureTimeMillis {
        largeRange.filter { it % 2 == 0 }
                  .map { it * 3 }
                  .filter { it > 100 }
                  .take(5)
                  .toList()
    }

    // ด้วย Sequence: lazy, ไม่สร้าง intermediate
    val seqTime = measureTimeMillis {
        largeRange.asSequence()
                  .filter { it % 2 == 0 }
                  .map { it * 3 }
                  .filter { it > 100 }
                  .take(5)
                  .toList()
    }

    println("List time: ${listTime}ms")
    println("Sequence time: ${seqTime}ms")
}

fun measureTimeMillis(block: () -> Unit): Long {
    val start = System.currentTimeMillis()
    block()
    return System.currentTimeMillis() - start
}
```

---

## Step 82: Advanced Map Operations

```kotlin
fun main() {
    // Advanced Map operations

    val wordCount = mutableMapOf<String, Int>()

    // นับคำใน text
    val text = "สวัสดี โลก สวัสดี kotlin kotlin kotlin โลก"
    text.split(" ").forEach { word ->
        wordCount[word] = (wordCount[word] ?: 0) + 1
    }
    println("word count: $wordCount")

    // getOrPut - ดึงค่า ถ้าไม่มีให้ใส่ค่า default
    val cache = mutableMapOf<String, List<Int>>()
    fun getCachedData(key: String): List<Int> {
        return cache.getOrPut(key) { 
            println("computing for $key...")
            (1..5).map { it * key.length }
        }
    }
    println(getCachedData("hello"))
    println(getCachedData("hello")) // ใช้ cache
    println(getCachedData("world"))

    // merge - รวม maps
    println("\n--- merge maps ---")
    val map1 = mapOf("a" to 1, "b" to 2, "c" to 3)
    val map2 = mapOf("b" to 20, "c" to 30, "d" to 40)

    val merged = (map1.entries + map2.entries)
        .groupBy({ it.key }, { it.value })
        .mapValues { it.value.sum() }
    println("merged: $merged")

    // filterKeys, filterValues
    val prices = mapOf("iPhone" to 35000, "AirPods" to 8000, "MacBook" to 65000, "iPad" to 25000)
    val expensive = prices.filter { (_, price) -> price > 20000 }
    println("\nราคาสูงกว่า 20000: $expensive")

    val appleProducts = prices.filterKeys { it.startsWith("i") }
    println("สินค้าที่ขึ้นต้นด้วย i: $appleProducts")

    // mapValues, mapKeys
    val withDiscount = prices.mapValues { (_, price) -> price * 0.9 }
    println("ลด 10%: $withDiscount")

    val lowercase = prices.mapKeys { (key, _) -> key.lowercase() }
    println("lowercase keys: $lowercase")

    // toSortedMap
    val sorted = prices.toSortedMap()
    println("sorted by key: $sorted")

    val sortedByValue = prices.entries
        .sortedBy { it.value }
        .associate { it.key to it.value }
    println("sorted by value: $sortedByValue")

    // any, all, none ใน Map
    println("\nมีสินค้าราคาเกิน 50000: ${prices.any { it.value > 50000 }}")
    println("ทุกสินค้าราคาเกิน 5000: ${prices.all { it.value > 5000 }}")
    println("ไม่มีสินค้าฟรี: ${prices.none { it.value == 0 }}")
}
```

---

## Step 83: Collection Transformation - แปลง Collection

```kotlin
data class Person(val name: String, val age: Int, val city: String)

fun main() {
    val people = listOf(
        Person("สมชาย", 25, "กรุงเทพ"),
        Person("สมหญิง", 30, "เชียงใหม่"),
        Person("สมศักดิ์", 22, "กรุงเทพ"),
        Person("สมใจ", 28, "ขอนแก่น"),
        Person("สมพร", 35, "กรุงเทพ"),
        Person("สมบัติ", 27, "เชียงใหม่")
    )

    // distinct - ลบข้อมูลซ้ำ
    println("--- distinct ---")
    val cities = people.map { it.city }
    println("cities: $cities")
    println("distinct cities: ${cities.distinct()}")

    val distinctBy = people.distinctBy { it.city }
    println("distinctBy city: ${distinctBy.map { "${it.name}(${it.city})" }}")

    // groupingBy - การจัดกลุ่มขั้นสูง
    println("\n--- groupingBy ---")
    val cityCount = people.groupingBy { it.city }.eachCount()
    println("คนในแต่ละเมือง: $cityCount")

    val cityAges = people.groupingBy { it.city }
        .fold(0) { acc, person -> acc + person.age }
    println("ผลรวมอายุตามเมือง: $cityAges")

    // ifEmpty, ifBlank
    println("\n--- ifEmpty ---")
    val emptyList = emptyList<String>()
    val withDefault = emptyList.ifEmpty { listOf("ค่าเริ่มต้น") }
    println("ifEmpty: $withDefault")

    // plus, minus operators
    println("\n--- plus/minus ---")
    val list1 = listOf(1, 2, 3)
    val list2 = list1 + 4 + 5
    println("list + elements: $list2")
    val list3 = list2 - 3
    println("list - 3: $list3")
    val list4 = list2 - listOf(2, 4)
    println("list - [2,4]: $list4")

    // toTypedArray
    println("\n--- toTypedArray ---")
    val array = list1.toTypedArray()
    println("array: ${array.contentToString()}")
    val backToList = array.toList()
    println("back to list: $backToList")

    // reversed
    val reversed = list1.reversed()
    println("reversed: $reversed")

    // shuffled (randomize)
    val original = listOf(1, 2, 3, 4, 5)
    val shuffled = original.shuffled()
    println("shuffled: $shuffled")

    // joinToString
    println("\n--- joinToString ---")
    val names = people.map { it.name }
    println(names.joinToString())
    println(names.joinToString(separator = " | "))
    println(names.joinToString(prefix = "[", postfix = "]"))
    println(names.joinToString(limit = 3, truncated = "..."))
}
```

---

## Step 84: Nested Collections

```kotlin
fun main() {
    // 2D List (Matrix)
    println("--- 2D Matrix ---")
    val matrix = listOf(
        listOf(1, 2, 3),
        listOf(4, 5, 6),
        listOf(7, 8, 9)
    )

    // print matrix
    matrix.forEach { row ->
        println(row.joinToString(" | "))
    }

    // เข้าถึงข้อมูล
    println("\nmatrix[1][2] = ${matrix[1][2]}") // 6

    // transpose matrix
    val transposed = (0 until matrix[0].size).map { col ->
        (0 until matrix.size).map { row ->
            matrix[row][col]
        }
    }
    println("\nTransposed:")
    transposed.forEach { row -> println(row.joinToString(" | ")) }

    // Map of Lists
    println("\n--- Map of Lists ---")
    val courseStudents = mapOf(
        "Kotlin" to listOf("สมชาย", "สมหญิง", "สมศักดิ์"),
        "Android" to listOf("สมใจ", "สมพร"),
        "Swift" to listOf("สมบัติ", "สมคิด", "สมนึก", "สมปอง")
    )

    courseStudents.forEach { (course, students) ->
        println("$course: ${students.size} คน - ${students.joinToString()}")
    }

    // หานักเรียนทั้งหมด (flatten)
    val allStudents = courseStudents.values.flatten()
    println("\nนักเรียนทั้งหมด: $allStudents")
    println("จำนวน: ${allStudents.size}")

    // List of Maps
    println("\n--- List of Maps ---")
    val records = listOf(
        mapOf("name" to "สมชาย", "score" to 85, "pass" to true),
        mapOf("name" to "สมหญิง", "score" to 92, "pass" to true),
        mapOf("name" to "สมศักดิ์", "score" to 55, "pass" to false)
    )

    val passingStudents = records.filter { it["pass"] == true }
    println("ผ่าน: ${passingStudents.map { it["name"] }}")

    val avgScore = records.map { it["score"] as Int }.average()
    println("คะแนนเฉลี่ย: $avgScore")

    // flatMap กับ nested
    val departments = mapOf(
        "IT" to listOf("Java", "Kotlin", "Python"),
        "Design" to listOf("Figma", "Photoshop"),
        "Marketing" to listOf("SEO", "Social Media", "Content")
    )

    val allSkills = departments.values.flatten()
    println("\nทักษะทั้งหมด: $allSkills")

    val skillCount = departments.flatMap { (_, skills) -> skills }
        .groupingBy { it }
        .eachCount()
    println("จำนวนแต่ละทักษะ: $skillCount")
}
```

---

## Step 85: Collection Best Practices

```kotlin
fun main() {
    // Best Practices สำหรับ Collections

    // 1. ใช้ immutable ก่อนเสมอ
    val immutable = listOf(1, 2, 3) // ดี
    val mutable = mutableListOf(1, 2, 3) // ใช้เมื่อจำเป็น

    // 2. ใช้ appropriate collection type
    // List - ต้องการลำดับ / อนุญาตซ้ำ
    val orderedList = listOf("ก", "ข", "ก", "ค")

    // Set - ต้องการ uniqueness
    val uniqueItems = setOf("ก", "ข", "ก", "ค") // {ก, ข, ค}

    // Map - ต้องการ key-value lookup
    val lookup = mapOf("th" to "ภาษาไทย", "en" to "English")

    // 3. ใช้ Sequence สำหรับ collection ใหญ่
    val big = (1..1000000).asSequence()
        .filter { it % 2 == 0 }
        .map { it * 2 }
        .take(10)
        .toList()
    println("big sequence: $big")

    // 4. ใช้ extension functions แทน loops
    val numbers = listOf(1, 2, 3, 4, 5, 6, 7, 8, 9, 10)

    // ไม่ดี
    val evenSquaresBad = mutableListOf<Int>()
    for (n in numbers) {
        if (n % 2 == 0) evenSquaresBad.add(n * n)
    }

    // ดี
    val evenSquaresGood = numbers.filter { it % 2 == 0 }.map { it * it }
    println("even squares: $evenSquaresGood")

    // 5. Chain operations อย่างมีประสิทธิภาพ
    data class Item(val name: String, val price: Int, val qty: Int)
    val items = listOf(
        Item("A", 100, 5),
        Item("B", 200, 3),
        Item("C", 50, 10),
        Item("D", 300, 2)
    )

    val topItems = items
        .filter { it.qty > 2 }
        .sortedByDescending { it.price * it.qty }
        .take(3)
        .map { "${it.name}: ${it.price * it.qty} บาท" }

    println("\nTop items:")
    topItems.forEach { println("  $it") }

    // 6. ใช้ elvis operator กับ null collections
    val maybeNull: List<Int>? = null
    val safe = maybeNull ?: emptyList()
    println("\nsafe: $safe")

    // 7. ใช้ orEmpty()
    val nullableList: List<String>? = null
    println("orEmpty: ${nullableList.orEmpty()}")

    val nullableMap: Map<String, Int>? = null
    println("map orEmpty: ${nullableMap.orEmpty()}")
}
```

---

## Step 86: Collection เขียน Custom Operations

```kotlin
// Extension functions สำหรับ Collections

// หาค่าเฉลี่ยของ List<Int>
fun List<Int>.average(): Double = if (isEmpty()) 0.0 else sum().toDouble() / size

// กรองและแปลงพร้อมกัน
fun <T, R : Any> List<T>.filterMap(predicate: (T) -> Boolean, transform: (T) -> R): List<R> {
    return filter(predicate).map(transform)
}

// นับคำในข้อความ
fun String.wordCount(): Map<String, Int> {
    return split("\\s+".toRegex())
        .filter { it.isNotEmpty() }
        .groupingBy { it }
        .eachCount()
}

// หาค่า mode (ค่าที่พบบ่อยที่สุด)
fun <T> List<T>.mode(): T? {
    return groupingBy { it }
        .eachCount()
        .maxByOrNull { it.value }
        ?.key
}

// batch processing
fun <T> List<T>.processBatches(batchSize: Int, process: (List<T>) -> Unit) {
    chunked(batchSize).forEach { batch ->
        process(batch)
    }
}

fun main() {
    val scores = listOf(85, 92, 78, 95, 88, 76, 91, 83, 89, 77)
    println("คะแนน: $scores")
    println("เฉลี่ย: ${scores.average()}")

    val highScores = scores.filterMap({ it >= 85 }, { "A: $it" })
    println("คะแนนสูง: $highScores")

    val text = "the quick brown fox jumps over the lazy dog the fox"
    val wordCounts = text.wordCount()
    println("\nนับคำ: $wordCounts")

    val mostCommon = wordCounts.entries.maxByOrNull { it.value }
    println("คำที่พบบ่อยที่สุด: ${mostCommon?.key} (${mostCommon?.value} ครั้ง)")

    val grades = listOf("A", "B", "A", "C", "B", "A", "B", "A")
    println("\nเกรดที่พบบ่อยที่สุด: ${grades.mode()}")

    println("\n--- batch processing ---")
    val items = (1..20).toList()
    items.processBatches(5) { batch ->
        println("batch: $batch, sum=${batch.sum()}")
    }

    // ใช้ run, let, also กับ collections
    println("\n--- scoped operations ---")
    val result = listOf(1, 2, 3, 4, 5)
        .also { println("original: $it") }
        .filter { it % 2 == 0 }
        .also { println("filtered: $it") }
        .map { it * 10 }
        .also { println("mapped: $it") }

    println("final: $result")
}
```

---

## Step 87: ใช้ Collections กับ Data Classes

```kotlin
data class Student(
    val id: Int,
    val name: String,
    val grade: Double,
    val subjects: List<String>
)

data class ClassRoom(
    val name: String,
    val students: List<Student>
)

fun ClassRoom.statistics() {
    println("=== $name Statistics ===")
    println("จำนวนนักเรียน: ${students.size}")
    println("คะแนนเฉลี่ย: ${"%.2f".format(students.map { it.grade }.average())}")
    println("คะแนนสูงสุด: ${students.maxOf { it.grade }} (${students.maxByOrNull { it.grade }?.name})")
    println("คะแนนต่ำสุด: ${students.minOf { it.grade }} (${students.minByOrNull { it.grade }?.name})")
    
    val gradeGroups = students.groupBy {
        when {
            it.grade >= 90 -> "A"
            it.grade >= 80 -> "B"
            it.grade >= 70 -> "C"
            it.grade >= 60 -> "D"
            else -> "F"
        }
    }
    println("การกระจายเกรด:")
    gradeGroups.entries.sortedBy { it.key }.forEach { (grade, studs) ->
        println("  $grade: ${studs.size} คน (${studs.map { it.name }.joinToString()})")
    }

    val allSubjects = students.flatMap { it.subjects }.distinct().sorted()
    println("วิชาที่เรียน: $allSubjects")

    val subjectPopularity = students.flatMap { it.subjects }
        .groupingBy { it }
        .eachCount()
        .entries
        .sortedByDescending { it.value }
    println("วิชายอดนิยม: ${subjectPopularity.take(3).map { "${it.key}(${it.value})" }}")
}

fun main() {
    val classroom = ClassRoom(
        name = "Kotlin 101",
        students = listOf(
            Student(1, "สมชาย", 85.5, listOf("Math", "Science", "Kotlin")),
            Student(2, "สมหญิง", 92.0, listOf("Science", "Kotlin", "English")),
            Student(3, "สมศักดิ์", 71.5, listOf("Math", "Kotlin")),
            Student(4, "สมใจ", 95.0, listOf("Math", "Science", "Kotlin", "English")),
            Student(5, "สมพร", 63.0, listOf("Kotlin", "English")),
            Student(6, "สมบัติ", 78.5, listOf("Math", "Science"))
        )
    )

    classroom.statistics()

    // ค้นหานักเรียน
    println("\n--- ค้นหา ---")
    val topStudents = classroom.students
        .filter { it.grade >= 85 }
        .sortedByDescending { it.grade }
    
    println("นักเรียนคะแนนสูง:")
    topStudents.forEach { println("  ${it.name}: ${it.grade}") }

    // นักเรียนที่เรียน Kotlin
    val kotlinStudents = classroom.students.filter { "Kotlin" in it.subjects }
    println("\nนักเรียนที่เรียน Kotlin: ${kotlinStudents.map { it.name }}")

    // อัพเดท (copy)
    val updatedStudent = classroom.students[0].copy(grade = 90.0)
    println("\nอัพเดทคะแนน: ${classroom.students[0].name} ${classroom.students[0].grade} -> ${updatedStudent.grade}")
}
```

---

## Step 88: Collection Performance Tips

```kotlin
fun main() {
    // Tips สำหรับ performance

    // 1. ระบุขนาดเริ่มต้นเมื่อรู้ขนาด
    val knownSize = ArrayList<Int>(1000)
    (1..1000).forEach { knownSize.add(it) }

    // 2. ใช้ indices แทน range for index access
    val list = listOf(1, 2, 3, 4, 5)
    
    // ดี
    for (i in list.indices) {
        print("${list[i]} ")
    }
    println()

    // ดีกว่า
    list.forEachIndexed { i, v -> print("$i:$v ") }
    println()

    // 3. ใช้ contains บน Set แทน List (O(1) vs O(n))
    val listForSearch = (1..10000).toList()
    val setForSearch = (1..10000).toSet()

    val searchValue = 9999

    val listTime = measureTimeMillis {
        repeat(10000) { listForSearch.contains(searchValue) }
    }
    val setTime = measureTimeMillis {
        repeat(10000) { setForSearch.contains(searchValue) }
    }
    println("List search: ${listTime}ms")
    println("Set search: ${setTime}ms")

    // 4. ระวัง toList() ใน loop ใหญ่
    val bigList = (1..100000).toList()
    
    // ไม่ดี: สร้าง list ใหม่ทุก iteration
    var resultBad = bigList
    repeat(5) {
        resultBad = resultBad.filter { it % 2 == 0 }
    }
    
    // ดีกว่า: ใช้ sequence
    val resultGood = bigList.asSequence()
        .filter { it % 2 == 0 }
        .filter { it % 3 == 0 }
        .filter { it % 5 == 0 }
        .toList()
    println("result size: ${resultGood.size}")

    // 5. ใช้ firstOrNull แทน filter().firstOrNull()
    // ไม่ดี: filter ทั้ง list ก่อน
    val first1 = bigList.filter { it > 50000 }.firstOrNull()
    // ดี: หยุดเมื่อเจอตัวแรก
    val first2 = bigList.firstOrNull { it > 50000 }
    println("first1: $first1, first2: $first2")

    // 6. ใช้ count{} แทน filter{}.size
    val countBad = bigList.filter { it % 2 == 0 }.size
    val countGood = bigList.count { it % 2 == 0 }
    println("count: $countBad == $countGood")
}

fun measureTimeMillis(block: () -> Unit): Long {
    val start = System.currentTimeMillis()
    block()
    return System.currentTimeMillis() - start
}
```

---

## Step 89: สรุป Collection Operators

```kotlin
fun main() {
    val data = listOf(
        Triple("สมชาย", 25, "กรุงเทพ"),
        Triple("สมหญิง", 30, "เชียงใหม่"),
        Triple("สมศักดิ์", 22, "กรุงเทพ"),
        Triple("สมใจ", 28, "ขอนแก่น"),
        Triple("สมพร", 35, "กรุงเทพ"),
        Triple("สมบัติ", 27, "เชียงใหม่"),
        Triple("สมคิด", 24, "ขอนแก่น"),
        Triple("สมนึก", 31, "กรุงเทพ")
    )

    // สรุปการใช้งาน operators ทั้งหมด
    println("=== สรุป Collection Operators ===\n")

    // Filtering
    println("--- Filtering ---")
    println("filter (อายุ>25): ${data.filter { it.second > 25 }.map { it.first }}")
    println("filterNot (ไม่ใช่กรุงเทพ): ${data.filterNot { it.third == "กรุงเทพ" }.map { it.first }}")
    val (young, old) = data.partition { it.second < 27 }
    println("partition <27: young=${young.map{it.first}}, old=${old.map{it.first}}")

    // Transformation
    println("\n--- Transformation ---")
    println("map (names): ${data.map { it.first }}")
    println("flatMap: ${data.flatMap { listOf(it.first, it.third) }.distinct()}")
    
    // Ordering
    println("\n--- Ordering ---")
    println("sortedBy age: ${data.sortedBy { it.second }.map { "${it.first}(${it.second})" }}")
    println("sortedByDescending: ${data.sortedByDescending { it.second }.take(3).map { it.first }}")

    // Grouping
    println("\n--- Grouping ---")
    val byCity = data.groupBy { it.third }
    byCity.forEach { (city, people) -> println("$city: ${people.size} คน") }

    // Aggregation
    println("\n--- Aggregation ---")
    println("count: ${data.count()}")
    println("count >25: ${data.count { it.second > 25 }}")
    println("avg age: ${"%.1f".format(data.map { it.second }.average())}")
    println("sum ages: ${data.sumOf { it.second }}")
    println("max age: ${data.maxOf { it.second }}")
    println("min age: ${data.minOf { it.second }}")

    // Finding
    println("\n--- Finding ---")
    println("find (กรุงเทพ): ${data.find { it.third == "กรุงเทพ" }?.first}")
    println("any (อายุ>30): ${data.any { it.second > 30 }}")
    println("all (อายุ>18): ${data.all { it.second > 18 }}")
    println("none (อายุ<20): ${data.none { it.second < 20 }}")

    // Set operations
    val bkk = data.filter { it.third == "กรุงเทพ" }.map { it.first }.toSet()
    val old2 = data.filter { it.second > 27 }.map { it.first }.toSet()
    println("\n--- Set Operations ---")
    println("กรุงเทพ: $bkk")
    println("อายุ>27: $old2")
    println("intersection: ${bkk.intersect(old2)}")
    println("union: ${bkk.union(old2)}")
    println("bkk - old: ${bkk.subtract(old2)}")
}
```

---

## Step 90: แบบฝึกหัดและโจทย์

```kotlin
// ============================================================
// แบบฝึกหัดที่ 1: วิเคราะห์ข้อมูลการขาย
// ============================================================

data class Sale(
    val date: String,
    val product: String,
    val category: String,
    val amount: Int,
    val price: Double
)

fun analyzesSales(sales: List<Sale>) {
    println("=== Sales Analysis ===")

    // รายได้รวม
    val totalRevenue = sales.sumOf { it.amount * it.price }
    println("รายได้รวม: ${"%.2f".format(totalRevenue)}")

    // สินค้าขายดีที่สุด
    val topProduct = sales.groupBy { it.product }
        .mapValues { (_, s) -> s.sumOf { it.amount } }
        .maxByOrNull { it.value }
    println("สินค้าขายดี: ${topProduct?.key} (${topProduct?.value} ชิ้น)")

    // รายได้ตาม category
    val revenueByCategory = sales.groupBy { it.category }
        .mapValues { (_, s) -> s.sumOf { it.amount * it.price } }
        .entries
        .sortedByDescending { it.value }
    println("รายได้ตาม category:")
    revenueByCategory.forEach { (cat, rev) ->
        println("  $cat: ${"%.2f".format(rev)}")
    }

    // ยอดขายรายวัน
    val dailySales = sales.groupBy { it.date }
        .mapValues { (_, s) -> s.sumOf { it.amount * it.price } }
        .entries
        .sortedBy { it.key }
    println("ยอดขายรายวัน:")
    dailySales.forEach { (date, rev) ->
        println("  $date: ${"%.2f".format(rev)}")
    }
}

fun main() {
    val sales = listOf(
        Sale("2024-01-01", "iPhone", "มือถือ", 5, 35000.0),
        Sale("2024-01-01", "AirPods", "อุปกรณ์เสริม", 10, 8000.0),
        Sale("2024-01-02", "MacBook", "คอมพิวเตอร์", 2, 65000.0),
        Sale("2024-01-02", "iPhone", "มือถือ", 8, 35000.0),
        Sale("2024-01-03", "iPad", "แท็บเล็ต", 4, 25000.0),
        Sale("2024-01-03", "AirPods", "อุปกรณ์เสริม", 15, 8000.0),
        Sale("2024-01-03", "MacBook", "คอมพิวเตอร์", 3, 65000.0)
    )

    analyzesSales(sales)

    // ============================================================
    // แบบฝึกหัดที่ 2: โจทย์ Kotlin Collections
    // ============================================================

    println("\n=== โจทย์ ===")

    // โจทย์ 1: หาเลขที่เป็นทั้ง prime และ มากกว่า 5
    fun isPrime(n: Int): Boolean {
        if (n < 2) return false
        return (2..Math.sqrt(n.toDouble()).toInt()).none { n % it == 0 }
    }
    val primes = (2..30).filter { isPrime(it) && it > 5 }
    println("Prime > 5 ถึง 30: $primes")

    // โจทย์ 2: หาคำที่ยาวที่สุดในแต่ละประโยค
    val sentences = listOf(
        "สวัสดี โลก ที่สวยงาม",
        "Kotlin เป็นภาษาที่น่าสนใจมาก",
        "Collections ช่วยจัดการข้อมูลได้ดี"
    )
    val longestWords = sentences.map { sentence ->
        sentence.split(" ").maxByOrNull { it.length }
    }
    println("คำยาวที่สุดในแต่ละประโยค: $longestWords")

    // โจทย์ 3: Anagram detection
    val words = listOf("listen", "silent", "enlist", "hello", "world", "dlrow")
    val anagramGroups = words.groupBy { it.toCharArray().sorted().joinToString("") }
        .values.filter { it.size > 1 }
    println("Anagram groups: $anagramGroups")

    // โจทย์ 4: สร้าง frequency table
    val text = "a b c a b a c d e a b c"
    val freq = text.split(" ")
        .groupingBy { it }
        .eachCount()
        .entries
        .sortedByDescending { it.value }
    println("Frequency table:")
    freq.forEach { (char, count) ->
        println("  $char: ${"█".repeat(count)} ($count)")
    }
}
```

---

## สรุปส่วนที่ 5 (Summary)

| Collection | Mutable | Immutable | ลักษณะ |
|------------|---------|-----------|--------|
| List | MutableList | List | เก็บลำดับ อนุญาตซ้ำ |
| Set | MutableSet | Set | ไม่ซ้ำ |
| Map | MutableMap | Map | key-value pairs |

### Operators สำคัญ

| Operator | หน้าที่ |
|----------|--------|
| `filter` | กรองตามเงื่อนไข |
| `map` | แปลงแต่ละ element |
| `flatMap` | map แล้ว flatten |
| `reduce/fold` | รวบรวมเป็นค่าเดียว |
| `groupBy` | จัดกลุ่ม |
| `sortedBy` | เรียงลำดับ |
| `any/all/none` | ตรวจสอบเงื่อนไข |
| `count` | นับ |
| `partition` | แบ่ง 2 กลุ่ม |
| `zip` | จับคู่ 2 lists |
| `distinct` | ลบซ้ำ |

---

[← Part 04: Control Flow](../part04/README.md) | [Part 06: OOP Basics →](../part06/README.md)
