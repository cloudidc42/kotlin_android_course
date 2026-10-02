# Part 98: Kotlin Interview Questions & Answers
## ขั้นตอนที่ 1926-1950

---

## ขั้นตอนที่ 1926: Core Kotlin Questions

```kotlin
// ============================================
// Q1: ความแตกต่างระหว่าง val และ var
// ============================================

val name = "Alice"    // Immutable reference (สามารถ mutate object ได้)
var age = 25          // Mutable reference

// val ไม่ได้หมายความว่า object เป็น immutable!
val list = mutableListOf(1, 2, 3)
list.add(4)  // ได้! เพราะ list object ยังสามารถ mutate ได้
// list = mutableListOf()  // ไม่ได้! เพราะ reference เป็น val

// ============================================
// Q2: data class คืออะไร
// ============================================

// data class ให้ automatic:
// - equals() / hashCode() - based on properties
// - toString() - ชื่อ class + properties
// - copy() - shallow copy
// - componentN() - destructuring

data class User(val id: Long, val name: String, val email: String)

val user1 = User(1, "Alice", "alice@test.com")
val user2 = user1.copy(name = "Bob")
val (id, name, email) = user1  // destructuring

// ============================================
// Q3: sealed class vs enum class
// ============================================

// enum: fixed values, ทุก instance มี type เดียวกัน
enum class Direction { NORTH, SOUTH, EAST, WEST }

// sealed: hierarchy, แต่ละ subclass มี state ของตัวเอง
sealed class Result<out T> {
    data class Success<T>(val data: T) : Result<T>()
    data class Error(val message: String, val code: Int) : Result<Nothing>()
    object Loading : Result<Nothing>()
}

// ============================================
// Q4: object declaration vs companion object
// ============================================

// object declaration: Singleton
object DatabaseConfig {
    val url = "jdbc:mysql://localhost/mydb"
    val maxConnections = 10
}

// companion object: static-like, tied to class
class UserFactory {
    companion object {
        fun create(name: String) = User(id = System.nanoTime(), name = name, email = "")
    }
}
UserFactory.create("Alice")  // เรียกโดยไม่ต้อง instantiate

// ============================================
// Q5: inline functions
// ============================================

// inline ทำให้ function body ถูก inline ตรงที่เรียกใช้
// ช่วยลด overhead ของ lambda allocation

inline fun measureTime(block: () -> Unit): Long {
    val start = System.nanoTime()
    block()
    return System.nanoTime() - start
}

// noinline: สำหรับ lambda ที่ไม่ต้องการ inline
inline fun doSomething(action: () -> Unit, noinline callback: () -> Unit) {
    action()  // inlined
    saveCallback(callback)  // not inlined (เก็บไว้ใช้ทีหลัง)
}

// crossinline: ไม่ให้ non-local return
inline fun runSafely(crossinline block: () -> Unit) {
    try {
        block()  // block ไม่สามารถทำ return@runSafely ได้
    } catch (e: Exception) {
        // handle
    }
}
```

---

## ขั้นตอนที่ 1927: Coroutines Interview Questions

```kotlin
// ============================================
// Q6: suspend function คืออะไร
// ============================================

// suspend function ถูก compile ให้รับ Continuation parameter
// สามารถ suspend (หยุด) การทำงานโดยไม่ block thread

suspend fun fetchUser(id: Long): User {
    delay(1000)  // suspend ไม่ block thread
    return api.getUser(id)
}

// Under the hood:
// fun fetchUser(id: Long, continuation: Continuation<User>): Any

// ============================================
// Q7: Dispatcher คืออะไร มีอะไรบ้าง
// ============================================

// Dispatcher กำหนดว่า coroutine ทำงานบน thread ไหน

// Dispatchers.Main - Main/UI thread (Android only)
// Dispatchers.IO - Network, disk I/O (thread pool 64+)
// Dispatchers.Default - CPU-intensive (thread pool = CPU count)
// Dispatchers.Unconfined - ทำงานบน thread ที่ resume

coroutineScope.launch(Dispatchers.IO) {
    val data = fetchFromNetwork()  // IO thread
    withContext(Dispatchers.Main) {
        updateUi(data)  // Main thread
    }
}

// ============================================
// Q8: ความแตกต่างระหว่าง launch และ async
// ============================================

// launch: Fire-and-forget, return Job
val job = scope.launch {
    doWork()  // ไม่ return value
}
job.cancel()

// async: Return Deferred (ใช้ await() เพื่อรับผลลัพธ์)
val deferred = scope.async {
    computeValue()  // return value
}
val result = deferred.await()

// Parallel execution
coroutineScope {
    val user = async { fetchUser(id) }
    val posts = async { fetchPosts(id) }
    Dashboard(user.await(), posts.await())
}

// ============================================
// Q9: ความแตกต่างระหว่าง coroutineScope และ supervisorScope
// ============================================

// coroutineScope: ถ้า child ล้มเหลว → cancel ทุก children
coroutineScope {
    launch { task1() }  // ถ้า task1 throw → task2 ถูก cancel
    launch { task2() }
}

// supervisorScope: ถ้า child ล้มเหลว → ไม่ cancel siblings
supervisorScope {
    launch { task1() }  // ถ้า task1 throw → task2 ยังทำงาน
    launch { task2() }
}

// ============================================
// Q10: structured concurrency คืออะไร
// ============================================

/*
Structured Concurrency:
1. Coroutine ต้องมี parent scope
2. Parent รอ children เสมอก่อน complete
3. Parent cancel → children ทุกตัวถูก cancel
4. Child fail → แจ้ง parent (ยกเว้น supervisorScope)

ประโยชน์:
- ไม่มี coroutine leak
- Error propagation ชัดเจน
- Lifecycle management อัตโนมัติ
*/
```

---

## ขั้นตอนที่ 1928: Android Architecture Questions

```kotlin
// ============================================
// Q11: LifecycleScope vs ViewModelScope
// ============================================

// lifecycleScope: tied to Activity/Fragment lifecycle
//   cancel เมื่อ Activity/Fragment destroyed

class MyFragment : Fragment() {
    override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
        // Cancelled when fragment's view is destroyed
        viewLifecycleOwner.lifecycleScope.launch {
            viewModel.data.collect { updateUi(it) }
        }
    }
}

// viewModelScope: tied to ViewModel lifecycle
//   cancel เมื่อ ViewModel cleared (Activity destroyed ครั้งสุดท้าย)

class MyViewModel : ViewModel() {
    init {
        viewModelScope.launch {
            fetchData()  // สามารถ survive configuration changes
        }
    }
}

// ============================================
// Q12: StateFlow vs LiveData
// ============================================

/*
LiveData:
- Lifecycle-aware (ไม่ emit เมื่อ lifecycle inactive)
- ต้องใช้บน Main thread
- Java-friendly
- setValue vs postValue

StateFlow:
- Coroutine-native
- Thread-safe
- ไม่ผูกกับ lifecycle โดย default (ใช้ collectAsStateWithLifecycle)
- Flow operators (map, filter, etc.)
- Testing ง่ายกว่า

เลือกใช้ StateFlow ในโปรเจกต์ใหม่
*/

// ============================================
// Q13: Single Activity Architecture
// ============================================

/*
Single Activity:
- 1 Activity + หลาย Composable/Fragment (Navigation)
- ข้อดี: ใช้ back stack เดียว, share ViewModel ง่าย
- Navigation Compose/Fragment ทำงานกับ pattern นี้ได้ดี

ปัญหาเดิม:
- หลาย Activity → ส่งข้อมูลผ่าน Intent (ยุ่งยาก)
- Back stack ซับซ้อน
- Share ViewModel ไม่ได้ระหว่าง Activities

Solution ปัจจุบัน:
- Navigation Compose ด้วย NavHost
- Shared ViewModel ใน NavGraph scope
*/

// ============================================
// Q14: Configuration Changes
// ============================================

/*
Configuration Change (เช่น หมุนจอ):
1. Activity destroyed + recreated
2. ViewModel ยังอยู่ (cleared เมื่อ user กด back จริงๆ)
3. StateFlow ใน ViewModel → data ไม่หาย

Handle ใน Compose:
- rememberSaveable → save ผ่าน configuration change AND process death
- remember → save เฉพาะ configuration change
- ViewModel → save data ผ่าน configuration change (ไม่ผ่าน process death)
- SavedStateHandle ใน ViewModel → save ผ่านทั้งคู่
*/
```

---

## ขั้นตอนที่ 1929: Performance Questions

```kotlin
// ============================================
// Q15: Main Thread Rule
// ============================================

/*
Rule: UI ต้องทำงานบน Main thread เท่านั้น
- ต้อง update View บน Main thread
- ต้องไม่ทำงานหนักบน Main thread (network, database, complex computation)

ANR (Application Not Responding):
- Activity ไม่ตอบสนอง 5 วินาที
- Broadcast Receiver ไม่ตอบสนอง 10 วินาที
- เกิดจาก: blocking Main thread

Prevention:
- ใช้ Coroutines + Dispatchers.IO สำหรับ I/O
- ใช้ Dispatchers.Default สำหรับ CPU-intensive
- ไม่ทำ network calls บน Main thread
*/

// ============================================
// Q16: Memory Leaks ใน Android
// ============================================

/*
Common Leaks:
1. Static reference ไปยัง Activity/Context
2. Non-static inner class (ถือ outer class reference)
3. Anonymous class / lambda ในกรณีที่ outlive owner
4. Unregistered listeners (BroadcastReceiver, etc.)
5. Handler โดยไม่ใช้ WeakReference

Prevention:
- ใช้ Application Context เมื่อไม่ต้องการ Activity
- ใช้ WeakReference สำหรับ Context ใน static context
- ใช้ viewLifecycleOwner.lifecycleScope แทน GlobalScope
- Unregister listeners ใน onDestroy/onPause
- LeakCanary ตรวจจับ leaks ระหว่าง development
*/

// ============================================
// Q17: Recomposition Optimization
// ============================================

/*
Compose Recomposition:
- Recompose เมื่อ state ที่อ่านเปลี่ยน
- ใช้ @Stable, @Immutable ให้ compiler รู้ว่า type ไม่เปลี่ยน
- remember() เพื่อ cache expensive computations
- derivedStateOf เพื่อ compute ค่าจาก state อื่น
- Key ที่ stable ใน LazyList
- Defer state reads ด้วย graphicsLayer, etc.

Debug:
- Layout Inspector → Recomposition counts
- Android Studio Profiler → frame drops
*/
```

---

*Part 98 จบแล้ว | ก่อนหน้า: [Part 97](../part97/README.md) | ถัดไป: [Part 99](../part99/README.md)*
