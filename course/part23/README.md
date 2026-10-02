# Part 23: Jetpack Compose พื้นฐาน
## ขั้นตอนที่ 546-575

---

## ขั้นตอนที่ 546: Jetpack Compose คืออะไร?

```kotlin
// Jetpack Compose = Modern Declarative UI Toolkit สำหรับ Android
// 
// ต่างจาก View System เดิม:
// View System: XML layout + Kotlin/Java code (Imperative)
// Compose: Pure Kotlin (Declarative)
//
// Declarative = บอก WHAT ให้แสดง ไม่ใช่ HOW ให้เปลี่ยน

// ============================================
// Composable Functions
// ============================================

import androidx.compose.foundation.layout.*
import androidx.compose.material3.*
import androidx.compose.runtime.*
import androidx.compose.ui.Alignment
import androidx.compose.ui.Modifier
import androidx.compose.ui.tooling.preview.Preview
import androidx.compose.ui.unit.dp

// @Composable คือ annotation ที่บอกว่าฟังก์ชันนี้ describe UI
@Composable
fun SimpleText() {
    Text(text = "Hello, Compose!")
}

// Composable ที่รับ parameter
@Composable
fun Greeting(name: String, modifier: Modifier = Modifier) {
    Text(
        text = "สวัสดี, $name!",
        modifier = modifier
    )
}

// Composable ที่ composable ซ้อนกัน
@Composable
fun UserCard(name: String, email: String) {
    Card(
        modifier = Modifier
            .fillMaxWidth()
            .padding(16.dp)
    ) {
        Column(
            modifier = Modifier.padding(16.dp)
        ) {
            Text(
                text = name,
                style = MaterialTheme.typography.titleLarge
            )
            Spacer(modifier = Modifier.height(4.dp))
            Text(
                text = email,
                style = MaterialTheme.typography.bodyMedium,
                color = MaterialTheme.colorScheme.onSurfaceVariant
            )
        }
    }
}

@Preview(showBackground = true)
@Composable
fun UserCardPreview() {
    MaterialTheme {
        UserCard(
            name = "สมชาย ใจดี",
            email = "somchai@example.com"
        )
    }
}
```

---

## ขั้นตอนที่ 547: State และ Recomposition

```kotlin
import androidx.compose.runtime.*

// ============================================
// State - ค่าที่เมื่อเปลี่ยน Compose จะ recompose
// ============================================

@Composable
fun CounterScreen() {
    // remember - เก็บค่าข้ามการ recompose
    // mutableIntStateOf - State ที่ track การเปลี่ยนแปลง
    var count by remember { mutableIntStateOf(0) }
    
    Column(
        horizontalAlignment = Alignment.CenterHorizontally,
        verticalArrangement = Arrangement.Center,
        modifier = Modifier
            .fillMaxSize()
            .padding(16.dp)
    ) {
        Text(
            text = "$count",
            style = MaterialTheme.typography.displayLarge
        )
        
        Spacer(modifier = Modifier.height(16.dp))
        
        Row(horizontalArrangement = Arrangement.spacedBy(8.dp)) {
            Button(onClick = { count-- }) { Text("-") }
            Button(onClick = { count++ }) { Text("+") }
        }
        
        Spacer(modifier = Modifier.height(8.dp))
        
        TextButton(onClick = { count = 0 }) {
            Text("Reset")
        }
    }
}

// ============================================
// State Hoisting - ยก State ขึ้นไปข้างบน
// ============================================

// Stateless Composable (ดีกว่า - testable, reusable)
@Composable
fun CounterDisplay(
    count: Int,
    onIncrement: () -> Unit,
    onDecrement: () -> Unit,
    onReset: () -> Unit,
    modifier: Modifier = Modifier
) {
    Column(
        horizontalAlignment = Alignment.CenterHorizontally,
        modifier = modifier.padding(16.dp)
    ) {
        Text(
            text = "$count",
            style = MaterialTheme.typography.displayLarge
        )
        
        Row(horizontalArrangement = Arrangement.spacedBy(8.dp)) {
            Button(onClick = onDecrement) { Text("-") }
            Button(onClick = onIncrement) { Text("+") }
        }
        
        TextButton(onClick = onReset) { Text("Reset") }
    }
}

// Stateful parent
@Composable
fun CounterScreenV2() {
    var count by remember { mutableIntStateOf(0) }
    
    CounterDisplay(
        count = count,
        onIncrement = { count++ },
        onDecrement = { count-- },
        onReset = { count = 0 }
    )
}

// ============================================
// Multiple States
// ============================================

@Composable
fun LoginForm() {
    var email by remember { mutableStateOf("") }
    var password by remember { mutableStateOf("") }
    var isPasswordVisible by remember { mutableStateOf(false) }
    var isLoading by remember { mutableStateOf(false) }
    var errorMessage by remember { mutableStateOf<String?>(null) }
    
    Column(
        modifier = Modifier
            .fillMaxWidth()
            .padding(24.dp),
        verticalArrangement = Arrangement.spacedBy(16.dp)
    ) {
        Text(
            text = "เข้าสู่ระบบ",
            style = MaterialTheme.typography.headlineMedium
        )
        
        OutlinedTextField(
            value = email,
            onValueChange = { email = it; errorMessage = null },
            label = { Text("Email") },
            modifier = Modifier.fillMaxWidth(),
            singleLine = true
        )
        
        OutlinedTextField(
            value = password,
            onValueChange = { password = it; errorMessage = null },
            label = { Text("Password") },
            modifier = Modifier.fillMaxWidth(),
            singleLine = true
            // visualTransformation สำหรับ password จะเรียนใน Part 24
        )
        
        errorMessage?.let { error ->
            Text(
                text = error,
                color = MaterialTheme.colorScheme.error,
                style = MaterialTheme.typography.bodySmall
            )
        }
        
        Button(
            onClick = {
                if (email.isBlank() || password.isBlank()) {
                    errorMessage = "กรุณากรอกข้อมูลให้ครบ"
                } else {
                    isLoading = true
                    // TODO: login logic
                }
            },
            modifier = Modifier.fillMaxWidth(),
            enabled = !isLoading
        ) {
            if (isLoading) {
                CircularProgressIndicator(
                    modifier = Modifier.size(16.dp),
                    strokeWidth = 2.dp
                )
            } else {
                Text("เข้าสู่ระบบ")
            }
        }
    }
}
```

---

## ขั้นตอนที่ 548: Layout Composables

```kotlin
// ============================================
// Column - วางแนวตั้ง
// ============================================

@Composable
fun ColumnExample() {
    Column(
        modifier = Modifier
            .fillMaxWidth()
            .padding(16.dp),
        horizontalAlignment = Alignment.CenterHorizontally,
        verticalArrangement = Arrangement.spacedBy(8.dp)
    ) {
        Text("รายการที่ 1")
        Text("รายการที่ 2")
        Text("รายการที่ 3")
    }
}

// ============================================
// Row - วางแนวนอน
// ============================================

@Composable
fun RowExample() {
    Row(
        modifier = Modifier
            .fillMaxWidth()
            .padding(16.dp),
        horizontalArrangement = Arrangement.SpaceBetween,
        verticalAlignment = Alignment.CenterVertically
    ) {
        Icon(imageVector = Icons.Default.Person, contentDescription = "User")
        Text("สมชาย ใจดี")
        Text("90 คะแนน", color = MaterialTheme.colorScheme.primary)
    }
}

// ============================================
// Box - วาง overlap กัน
// ============================================

@Composable
fun BoxExample() {
    Box(
        modifier = Modifier
            .size(200.dp)
            .background(MaterialTheme.colorScheme.primaryContainer)
    ) {
        // Bottom left
        Text(
            text = "Bottom Start",
            modifier = Modifier.align(Alignment.BottomStart).padding(8.dp)
        )
        
        // Center
        Text(
            text = "Center",
            modifier = Modifier.align(Alignment.Center)
        )
        
        // Top right
        Text(
            text = "Top End",
            modifier = Modifier.align(Alignment.TopEnd).padding(8.dp)
        )
    }
}

// ============================================
// Modifier
// ============================================

@Composable
fun ModifierExample() {
    Box(
        modifier = Modifier
            .fillMaxWidth()
            .height(200.dp)
            .background(MaterialTheme.colorScheme.surface)
            .padding(16.dp)
            .border(2.dp, MaterialTheme.colorScheme.primary)
            .clip(RoundedCornerShape(8.dp))
            .clickable { /* handle click */ }
    ) {
        Text("Modifier chain", modifier = Modifier.align(Alignment.Center))
    }
}

// ============================================
// Weight (Flex)
// ============================================

@Composable
fun WeightExample() {
    Row(modifier = Modifier.fillMaxWidth()) {
        // แบ่งพื้นที่ตาม weight
        Box(
            modifier = Modifier
                .weight(1f)  // 1 ส่วน
                .height(50.dp)
                .background(MaterialTheme.colorScheme.primaryContainer)
        )
        Box(
            modifier = Modifier
                .weight(2f)  // 2 ส่วน
                .height(50.dp)
                .background(MaterialTheme.colorScheme.secondaryContainer)
        )
        Box(
            modifier = Modifier
                .weight(1f)  // 1 ส่วน
                .height(50.dp)
                .background(MaterialTheme.colorScheme.tertiaryContainer)
        )
    }
}
```

---

## ขั้นตอนที่ 549: Material 3 Components

```kotlin
import androidx.compose.material.icons.Icons
import androidx.compose.material.icons.filled.*

// ============================================
// Scaffold - โครงสร้างหลักของหน้าจอ
// ============================================

@Composable
fun MainScreen() {
    var selectedItem by remember { mutableIntStateOf(0) }
    
    Scaffold(
        // Top App Bar
        topBar = {
            TopAppBar(
                title = { Text("My App") },
                navigationIcon = {
                    IconButton(onClick = { /* open drawer */ }) {
                        Icon(Icons.Default.Menu, contentDescription = "Menu")
                    }
                },
                actions = {
                    IconButton(onClick = { /* search */ }) {
                        Icon(Icons.Default.Search, contentDescription = "Search")
                    }
                    IconButton(onClick = { /* more */ }) {
                        Icon(Icons.Default.MoreVert, contentDescription = "More")
                    }
                },
                colors = TopAppBarDefaults.topAppBarColors(
                    containerColor = MaterialTheme.colorScheme.primaryContainer
                )
            )
        },
        
        // Bottom Navigation
        bottomBar = {
            NavigationBar {
                val items = listOf("หน้าแรก", "ค้นหา", "โปรไฟล์")
                val icons = listOf(Icons.Default.Home, Icons.Default.Search, Icons.Default.Person)
                
                items.forEachIndexed { index, item ->
                    NavigationBarItem(
                        icon = { Icon(icons[index], contentDescription = item) },
                        label = { Text(item) },
                        selected = selectedItem == index,
                        onClick = { selectedItem = index }
                    )
                }
            }
        },
        
        // FAB
        floatingActionButton = {
            FloatingActionButton(onClick = { /* add */ }) {
                Icon(Icons.Default.Add, contentDescription = "Add")
            }
        }
    ) { paddingValues ->
        // Content
        Box(
            modifier = Modifier
                .fillMaxSize()
                .padding(paddingValues)
        ) {
            Text(
                text = when (selectedItem) {
                    0 -> "หน้าแรก"
                    1 -> "ค้นหา"
                    2 -> "โปรไฟล์"
                    else -> ""
                },
                modifier = Modifier.align(Alignment.Center),
                style = MaterialTheme.typography.headlineMedium
            )
        }
    }
}

// ============================================
// Buttons
// ============================================

@Composable
fun ButtonExamples() {
    Column(
        modifier = Modifier.padding(16.dp),
        verticalArrangement = Arrangement.spacedBy(8.dp)
    ) {
        Button(onClick = {}) { Text("Filled Button") }
        OutlinedButton(onClick = {}) { Text("Outlined Button") }
        TextButton(onClick = {}) { Text("Text Button") }
        FilledTonalButton(onClick = {}) { Text("Tonal Button") }
        ElevatedButton(onClick = {}) { Text("Elevated Button") }
        
        // Button with Icon
        Button(onClick = {}) {
            Icon(Icons.Default.Download, contentDescription = null)
            Spacer(modifier = Modifier.width(8.dp))
            Text("Download")
        }
        
        // Icon Button
        IconButton(onClick = {}) {
            Icon(Icons.Default.Favorite, contentDescription = "Favorite")
        }
        
        // Disabled button
        Button(onClick = {}, enabled = false) { Text("Disabled") }
    }
}

// ============================================
// Cards
// ============================================

@Composable
fun CardExamples() {
    Column(
        modifier = Modifier.padding(16.dp),
        verticalArrangement = Arrangement.spacedBy(16.dp)
    ) {
        // Filled Card
        Card(
            modifier = Modifier.fillMaxWidth(),
            onClick = {}
        ) {
            Column(modifier = Modifier.padding(16.dp)) {
                Text("Filled Card", style = MaterialTheme.typography.titleMedium)
                Text("Card content", style = MaterialTheme.typography.bodyMedium)
            }
        }
        
        // Outlined Card
        OutlinedCard(modifier = Modifier.fillMaxWidth()) {
            Column(modifier = Modifier.padding(16.dp)) {
                Text("Outlined Card", style = MaterialTheme.typography.titleMedium)
            }
        }
        
        // Elevated Card
        ElevatedCard(modifier = Modifier.fillMaxWidth()) {
            Column(modifier = Modifier.padding(16.dp)) {
                Text("Elevated Card", style = MaterialTheme.typography.titleMedium)
            }
        }
    }
}

// ============================================
// Text Fields
// ============================================

@Composable
fun TextFieldExamples() {
    var text1 by remember { mutableStateOf("") }
    var text2 by remember { mutableStateOf("") }
    
    Column(
        modifier = Modifier.padding(16.dp),
        verticalArrangement = Arrangement.spacedBy(16.dp)
    ) {
        // Filled TextField
        TextField(
            value = text1,
            onValueChange = { text1 = it },
            label = { Text("Filled") },
            modifier = Modifier.fillMaxWidth()
        )
        
        // Outlined TextField
        OutlinedTextField(
            value = text2,
            onValueChange = { text2 = it },
            label = { Text("Outlined") },
            modifier = Modifier.fillMaxWidth(),
            leadingIcon = { Icon(Icons.Default.Email, null) },
            trailingIcon = {
                if (text2.isNotEmpty()) {
                    IconButton(onClick = { text2 = "" }) {
                        Icon(Icons.Default.Clear, null)
                    }
                }
            },
            supportingText = { Text("กรอก email ของคุณ") }
        )
    }
}

// ============================================
// Dialogs
// ============================================

@Composable
fun DialogExample() {
    var showDialog by remember { mutableStateOf(false) }
    
    Button(onClick = { showDialog = true }) {
        Text("แสดง Dialog")
    }
    
    if (showDialog) {
        AlertDialog(
            onDismissRequest = { showDialog = false },
            title = { Text("ยืนยันการลบ") },
            text = { Text("คุณต้องการลบข้อมูลนี้หรือไม่? การดำเนินการนี้ไม่สามารถย้อนกลับได้") },
            confirmButton = {
                TextButton(onClick = { showDialog = false }) {
                    Text("ยืนยัน", color = MaterialTheme.colorScheme.error)
                }
            },
            dismissButton = {
                TextButton(onClick = { showDialog = false }) {
                    Text("ยกเลิก")
                }
            }
        )
    }
}
```

---

## ขั้นตอนที่ 550: LazyColumn - แสดงรายการจำนวนมาก

```kotlin
import androidx.compose.foundation.lazy.*
import androidx.compose.foundation.lazy.grid.*

data class Product(
    val id: Int,
    val name: String,
    val price: Double,
    val category: String,
    val imageUrl: String = ""
)

// ============================================
// LazyColumn - เหมือน RecyclerView
// ============================================

@Composable
fun ProductList(products: List<Product>) {
    LazyColumn(
        modifier = Modifier.fillMaxSize(),
        contentPadding = PaddingValues(16.dp),
        verticalArrangement = Arrangement.spacedBy(8.dp)
    ) {
        // Header
        item {
            Text(
                "สินค้าทั้งหมด (${products.size} รายการ)",
                style = MaterialTheme.typography.titleLarge
            )
            Spacer(modifier = Modifier.height(8.dp))
        }
        
        // Items
        items(
            items = products,
            key = { it.id }  // stable key สำหรับ performance
        ) { product ->
            ProductItem(product = product)
        }
        
        // Footer
        item {
            Box(
                modifier = Modifier
                    .fillMaxWidth()
                    .padding(16.dp),
                contentAlignment = Alignment.Center
            ) {
                Text("โหลดแล้ว ${products.size} รายการ")
            }
        }
    }
}

@Composable
fun ProductItem(
    product: Product,
    modifier: Modifier = Modifier
) {
    Card(
        modifier = modifier.fillMaxWidth(),
        onClick = { /* navigate to detail */ }
    ) {
        Row(
            modifier = Modifier
                .fillMaxWidth()
                .padding(12.dp),
            verticalAlignment = Alignment.CenterVertically
        ) {
            // Image placeholder
            Box(
                modifier = Modifier
                    .size(80.dp)
                    .background(
                        MaterialTheme.colorScheme.surfaceVariant,
                        RoundedCornerShape(8.dp)
                    ),
                contentAlignment = Alignment.Center
            ) {
                Icon(
                    imageVector = Icons.Default.ShoppingCart,
                    contentDescription = null,
                    tint = MaterialTheme.colorScheme.onSurfaceVariant
                )
            }
            
            Spacer(modifier = Modifier.width(12.dp))
            
            Column(modifier = Modifier.weight(1f)) {
                Text(
                    text = product.name,
                    style = MaterialTheme.typography.titleMedium,
                    maxLines = 2,
                    overflow = TextOverflow.Ellipsis
                )
                Spacer(modifier = Modifier.height(4.dp))
                Text(
                    text = product.category,
                    style = MaterialTheme.typography.bodySmall,
                    color = MaterialTheme.colorScheme.onSurfaceVariant
                )
                Spacer(modifier = Modifier.height(4.dp))
                Text(
                    text = "฿${"%.2f".format(product.price)}",
                    style = MaterialTheme.typography.titleSmall,
                    color = MaterialTheme.colorScheme.primary
                )
            }
            
            IconButton(onClick = { /* add to cart */ }) {
                Icon(Icons.Default.Add, contentDescription = "Add to cart")
            }
        }
    }
}

// ============================================
// LazyVerticalGrid - Grid layout
// ============================================

@Composable
fun ProductGrid(products: List<Product>) {
    LazyVerticalGrid(
        columns = GridCells.Fixed(2),
        modifier = Modifier.fillMaxSize(),
        contentPadding = PaddingValues(16.dp),
        horizontalArrangement = Arrangement.spacedBy(8.dp),
        verticalArrangement = Arrangement.spacedBy(8.dp)
    ) {
        items(products, key = { it.id }) { product ->
            ProductGridItem(product)
        }
    }
}

@Composable
fun ProductGridItem(product: Product) {
    Card(
        modifier = Modifier.fillMaxWidth(),
        onClick = {}
    ) {
        Column {
            Box(
                modifier = Modifier
                    .fillMaxWidth()
                    .height(150.dp)
                    .background(MaterialTheme.colorScheme.surfaceVariant),
                contentAlignment = Alignment.Center
            ) {
                Icon(Icons.Default.ShoppingCart, null, modifier = Modifier.size(48.dp))
            }
            
            Column(modifier = Modifier.padding(12.dp)) {
                Text(
                    text = product.name,
                    style = MaterialTheme.typography.bodyMedium,
                    maxLines = 2,
                    overflow = TextOverflow.Ellipsis
                )
                Text(
                    text = "฿${"%.0f".format(product.price)}",
                    style = MaterialTheme.typography.titleSmall,
                    color = MaterialTheme.colorScheme.primary
                )
            }
        }
    }
}

@Preview(showBackground = true)
@Composable
fun ProductListPreview() {
    val sampleProducts = (1..10).map { i ->
        Product(i, "สินค้า #$i", (100..10000).random().toDouble(), "หมวดหมู่ ${(i % 3) + 1}")
    }
    
    MaterialTheme {
        ProductList(sampleProducts)
    }
}
```

---

## ขั้นตอนที่ 551: Side Effects ใน Compose

```kotlin
import androidx.compose.runtime.*

// ============================================
// LaunchedEffect - รัน code ครั้งเดียว (หรือเมื่อ key เปลี่ยน)
// ============================================

@Composable
fun TimerScreen() {
    var seconds by remember { mutableIntStateOf(0) }
    var isRunning by remember { mutableStateOf(false) }
    
    // LaunchedEffect รัน coroutine ที่ผูกกับ lifecycle ของ Composable
    // key = isRunning: restart เมื่อ isRunning เปลี่ยน
    LaunchedEffect(isRunning) {
        if (isRunning) {
            while (true) {
                delay(1000)
                seconds++
            }
        }
    }
    
    Column(
        modifier = Modifier
            .fillMaxSize()
            .padding(32.dp),
        horizontalAlignment = Alignment.CenterHorizontally,
        verticalArrangement = Arrangement.Center
    ) {
        val minutes = seconds / 60
        val secs = seconds % 60
        
        Text(
            text = "%02d:%02d".format(minutes, secs),
            style = MaterialTheme.typography.displayLarge
        )
        
        Spacer(modifier = Modifier.height(24.dp))
        
        Row(horizontalArrangement = Arrangement.spacedBy(16.dp)) {
            Button(onClick = { isRunning = !isRunning }) {
                Text(if (isRunning) "หยุด" else "เริ่ม")
            }
            OutlinedButton(onClick = { seconds = 0; isRunning = false }) {
                Text("Reset")
            }
        }
    }
}

// ============================================
// DisposableEffect - cleanup เมื่อ Composable ออก
// ============================================

@Composable
fun NetworkAwareScreen() {
    var isConnected by remember { mutableStateOf(true) }
    
    // จำลอง network listener
    // ใน Android จริงจะใช้ ConnectivityManager
    DisposableEffect(Unit) {
        println("เริ่ม network listener")
        
        // Cleanup
        onDispose {
            println("หยุด network listener")
        }
    }
    
    Scaffold { padding ->
        Box(
            modifier = Modifier
                .fillMaxSize()
                .padding(padding),
            contentAlignment = Alignment.Center
        ) {
            Text(
                text = if (isConnected) "มีการเชื่อมต่อ ✅" else "ไม่มีการเชื่อมต่อ ❌",
                style = MaterialTheme.typography.headlineMedium,
                color = if (isConnected) MaterialTheme.colorScheme.primary
                        else MaterialTheme.colorScheme.error
            )
        }
    }
}

// ============================================
// SideEffect - รัน code หลัง recompose ทุกครั้ง
// ============================================

@Composable
fun AnalyticsScreen(screenName: String) {
    // รัน analytics ทุกครั้งที่ screenName เปลี่ยน
    SideEffect {
        println("Analytics: view $screenName")
        // logEvent("screen_view", screenName)
    }
    
    Text("Screen: $screenName")
}

// ============================================
// rememberCoroutineScope - scope ที่ผูกกับ Composable
// ============================================

@Composable
fun SnackbarScreen() {
    val scope = rememberCoroutineScope()
    val snackbarHostState = remember { SnackbarHostState() }
    
    Scaffold(
        snackbarHost = { SnackbarHost(snackbarHostState) }
    ) { padding ->
        Column(
            modifier = Modifier
                .fillMaxSize()
                .padding(padding)
                .padding(16.dp),
            verticalArrangement = Arrangement.Center,
            horizontalAlignment = Alignment.CenterHorizontally
        ) {
            Button(onClick = {
                scope.launch {
                    val result = snackbarHostState.showSnackbar(
                        message = "บันทึกสำเร็จ!",
                        actionLabel = "ยกเลิก",
                        duration = SnackbarDuration.Short
                    )
                    when (result) {
                        SnackbarResult.ActionPerformed -> println("User clicked Undo")
                        SnackbarResult.Dismissed -> println("Snackbar dismissed")
                    }
                }
            }) {
                Text("แสดง Snackbar")
            }
        }
    }
}
```

---

## แบบฝึกหัด Part 23

```kotlin
// แบบฝึกหัดที่ 1: Shopping List App
// สร้าง Composable สำหรับ:
// - แสดงรายการสินค้า
// - เพิ่มสินค้าใหม่
// - ลบสินค้า (swipe หรือ checkbox)

data class ShoppingItem(
    val id: Int,
    val name: String,
    val quantity: Int = 1,
    val isChecked: Boolean = false
)

@Composable
fun ShoppingListApp() {
    // TODO: implement
    var items by remember { mutableStateOf(listOf<ShoppingItem>()) }
    var newItemText by remember { mutableStateOf("") }
    var nextId by remember { mutableIntStateOf(1) }
    
    Scaffold(
        topBar = {
            TopAppBar(title = { Text("รายการช้อปปิ้ง") })
        }
    ) { padding ->
        Column(
            modifier = Modifier
                .fillMaxSize()
                .padding(padding)
                .padding(16.dp)
        ) {
            // Input row
            Row(
                modifier = Modifier.fillMaxWidth(),
                horizontalArrangement = Arrangement.spacedBy(8.dp)
            ) {
                OutlinedTextField(
                    value = newItemText,
                    onValueChange = { newItemText = it },
                    label = { Text("รายการใหม่") },
                    modifier = Modifier.weight(1f),
                    singleLine = true
                )
                
                Button(
                    onClick = {
                        if (newItemText.isNotBlank()) {
                            items = items + ShoppingItem(nextId++, newItemText.trim())
                            newItemText = ""
                        }
                    }
                ) {
                    Icon(Icons.Default.Add, contentDescription = "Add")
                }
            }
            
            Spacer(modifier = Modifier.height(16.dp))
            
            // TODO: แสดงรายการ
            // - checkbox เพื่อ toggle isChecked
            // - ชื่อสินค้า (ขีดทับถ้า checked)
            // - ปุ่มลบ
        }
    }
}

// แบบฝึกหัดที่ 2: Profile Card
@Composable
fun ProfileCard(
    name: String,
    role: String,
    bio: String,
    followerCount: Int,
    followingCount: Int,
    onFollowClick: () -> Unit
) {
    // TODO: สร้าง Profile Card ที่สวยงาม
}
```

---

*Part 23 จบแล้ว | ก่อนหน้า: [Part 22](../part22/README.md) | ถัดไป: [Part 24](../part24/README.md)*
