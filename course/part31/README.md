# Part 31: Jetpack Compose เบื้องต้น
## ขั้นตอนที่ 551-575

---

## ขั้นตอนที่ 551: ทำความรู้จัก Jetpack Compose

```
Jetpack Compose คืออะไร?
- UI toolkit ใหม่ของ Android (แทนที่ XML layouts)
- Declarative: บอก "อยากได้ UI แบบไหน" ไม่ใช่ "ทำ UI ยังไง"
- 100% Kotlin
- ทำงานกับ Android lifecycle อัตโนมัติ

Declarative vs Imperative:

Imperative (XML + findViewById):
- สร้าง View
- หา View ด้วย findViewById
- เรียก textView.text = "Hello"
- จัดการ state ด้วยตัวเอง

Declarative (Compose):
- ประกาศ UI ตาม state
- เมื่อ state เปลี่ยน → UI เปลี่ยนอัตโนมัติ
- ไม่ต้อง mutate View โดยตรง
```

---

## ขั้นตอนที่ 552: @Composable Function

```kotlin
import androidx.compose.material3.Text
import androidx.compose.runtime.Composable

// ทุก UI component ใน Compose คือ @Composable function
@Composable
fun Greeting(name: String) {
    Text(text = "สวัสดี $name!")
}

// เรียกใช้ใน Composable อื่น
@Composable
fun MyApp() {
    Greeting(name = "โลก")
}

// MainActivity
class MainActivity : ComponentActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        
        setContent {
            // เริ่ม Compose content ที่นี่
            MyApp()
        }
    }
}
```

---

## ขั้นตอนที่ 553: Row, Column, Box

```kotlin
@Composable
fun LayoutDemo() {
    
    // Column: เรียงแนวตั้ง (เหมือน LinearLayout vertical)
    Column(
        verticalArrangement = Arrangement.spacedBy(8.dp),
        horizontalAlignment = Alignment.CenterHorizontally
    ) {
        Text("รายการที่ 1")
        Text("รายการที่ 2")
        Text("รายการที่ 3")
    }
    
    // Row: เรียงแนวนอน (เหมือน LinearLayout horizontal)
    Row(
        horizontalArrangement = Arrangement.SpaceBetween,
        verticalAlignment = Alignment.CenterVertically,
        modifier = Modifier.fillMaxWidth()
    ) {
        Text("ซ้าย")
        Text("กลาง")
        Text("ขวา")
    }
    
    // Box: ซ้อนกัน (เหมือน FrameLayout)
    Box(
        contentAlignment = Alignment.Center,
        modifier = Modifier.size(100.dp)
    ) {
        Box(
            modifier = Modifier
                .fillMaxSize()
                .background(Color.Blue)
        )
        Text("กลาง", color = Color.White)
    }
}
```

---

## ขั้นตอนที่ 554: Modifier

```kotlin
@Composable
fun ModifierDemo() {
    // Modifier ปรับแต่ง UI element
    // สำคัญ: ลำดับ modifier มีผล!
    
    Text(
        text = "Hello Compose",
        modifier = Modifier
            .padding(16.dp)             // padding รอบนอก
            .background(Color.Yellow)   // พื้นหลัง
            .padding(8.dp)              // padding ด้านใน
            .fillMaxWidth()             // กว้างเต็มจอ
            .height(50.dp)             // สูง 50dp
            .clickable { /* ทำอะไรบางอย่าง */ }
            .alpha(0.8f)               // โปร่งใส 80%
    )
    
    // size modifier
    Box(
        modifier = Modifier
            .size(100.dp)    // กว้าง × สูง เท่ากัน
            // หรือ
            .width(150.dp)
            .height(80.dp)
            // หรือ
            .fillMaxWidth()  // เต็มความกว้างของ parent
            .fillMaxHeight(0.5f)  // 50% ของความสูง parent
    )
}
```

---

## ขั้นตอนที่ 555: State ใน Compose

```kotlin
import androidx.compose.runtime.*

@Composable
fun CounterScreen() {
    // remember: เก็บ state ระหว่าง recomposition
    // mutableStateOf: สร้าง observable state
    var count by remember { mutableStateOf(0) }
    
    Column(
        horizontalAlignment = Alignment.CenterHorizontally,
        verticalArrangement = Arrangement.Center,
        modifier = Modifier.fillMaxSize()
    ) {
        Text(
            text = "นับ: $count",
            style = MaterialTheme.typography.headlineLarge
        )
        
        Spacer(modifier = Modifier.height(16.dp))
        
        Row(horizontalArrangement = Arrangement.spacedBy(16.dp)) {
            Button(onClick = { count-- }) {
                Text("-")
            }
            
            Button(onClick = { count++ }) {
                Text("+")
            }
        }
        
        Button(onClick = { count = 0 }) {
            Text("รีเซ็ต")
        }
    }
}
```

---

*Part 31 จบแล้ว | ก่อนหน้า: [Part 30](../part30/README.md) | ถัดไป: [Part 32](../part32/README.md)*
