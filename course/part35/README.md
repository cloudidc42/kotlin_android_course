# Part 35: Compose - Material 3 Components (Card, TopAppBar, NavigationBar)
## ขั้นตอนที่ 651-675

---

## ขั้นตอนที่ 651: Card Component แบบต่างๆ

Card ใน Material 3 มี 3 แบบ: Filled (ค่าเริ่มต้น), Elevated, Outlined

```kotlin
import androidx.compose.foundation.clickable
import androidx.compose.foundation.layout.*
import androidx.compose.material3.*
import androidx.compose.runtime.*
import androidx.compose.ui.Modifier
import androidx.compose.ui.unit.dp

@Composable
fun CardExamples() {
    Column(
        verticalArrangement = Arrangement.spacedBy(12.dp),
        modifier = Modifier.padding(16.dp)
    ) {
        // Filled Card (ค่าเริ่มต้น)
        Card(
            modifier = Modifier.fillMaxWidth()
        ) {
            Column(modifier = Modifier.padding(16.dp)) {
                Text("Filled Card", style = MaterialTheme.typography.titleMedium)
                Text("เนื้อหาใน card", style = MaterialTheme.typography.bodyMedium)
            }
        }

        // Elevated Card
        ElevatedCard(
            modifier = Modifier.fillMaxWidth(),
            elevation = CardDefaults.elevatedCardElevation(defaultElevation = 6.dp)
        ) {
            Column(modifier = Modifier.padding(16.dp)) {
                Text("Elevated Card", style = MaterialTheme.typography.titleMedium)
                Text("มีเงา elevation", style = MaterialTheme.typography.bodyMedium)
            }
        }

        // Outlined Card
        OutlinedCard(
            modifier = Modifier.fillMaxWidth()
        ) {
            Column(modifier = Modifier.padding(16.dp)) {
                Text("Outlined Card", style = MaterialTheme.typography.titleMedium)
                Text("มีกรอบ", style = MaterialTheme.typography.bodyMedium)
            }
        }

        // Clickable Card
        var isSelected by remember { mutableStateOf(false) }
        Card(
            modifier = Modifier
                .fillMaxWidth()
                .clickable { isSelected = !isSelected },
            colors = CardDefaults.cardColors(
                containerColor = if (isSelected)
                    MaterialTheme.colorScheme.primaryContainer
                else
                    MaterialTheme.colorScheme.surface
            )
        ) {
            Row(
                modifier = Modifier.padding(16.dp),
                horizontalArrangement = Arrangement.SpaceBetween
            ) {
                Text("คลิกเพื่อเลือก")
                if (isSelected) {
                    Icon(Icons.Default.Check, contentDescription = "เลือกแล้ว")
                }
            }
        }
    }
}
```

---

## ขั้นตอนที่ 652: TopAppBar แบบต่างๆ

Material 3 มี TopAppBar 4 แบบ: Small, Center-aligned, Medium, Large

```kotlin
import androidx.compose.material.icons.Icons
import androidx.compose.material.icons.filled.*
import androidx.compose.material3.*
import androidx.compose.runtime.*
import androidx.compose.ui.Modifier

@OptIn(ExperimentalMaterial3Api::class)
@Composable
fun TopAppBarExamples() {
    // Small TopAppBar (พื้นฐาน)
    Scaffold(
        topBar = {
            TopAppBar(
                title = { Text("หน้าแรก") },
                navigationIcon = {
                    IconButton(onClick = { /* back */ }) {
                        Icon(Icons.Default.ArrowBack, contentDescription = "กลับ")
                    }
                },
                actions = {
                    IconButton(onClick = { /* search */ }) {
                        Icon(Icons.Default.Search, contentDescription = "ค้นหา")
                    }
                    IconButton(onClick = { /* more */ }) {
                        Icon(Icons.Default.MoreVert, contentDescription = "เพิ่มเติม")
                    }
                }
            )
        }
    ) { paddingValues ->
        // เนื้อหาหน้า
    }
}

@OptIn(ExperimentalMaterial3Api::class)
@Composable
fun CenterAlignedTopAppBarExample() {
    Scaffold(
        topBar = {
            CenterAlignedTopAppBar(
                title = { Text("ตั้งค่า") },
                navigationIcon = {
                    IconButton(onClick = { }) {
                        Icon(Icons.Default.Close, contentDescription = "ปิด")
                    }
                },
                actions = {
                    TextButton(onClick = { }) {
                        Text("บันทึก")
                    }
                }
            )
        }
    ) { paddingValues ->
        // เนื้อหา
    }
}

@OptIn(ExperimentalMaterial3Api::class)
@Composable
fun CollapsingTopAppBarExample() {
    val scrollBehavior = TopAppBarDefaults.exitUntilCollapsedScrollBehavior()

    Scaffold(
        topBar = {
            // Large TopAppBar - ยุบได้เมื่อ scroll
            LargeTopAppBar(
                title = { Text("รายการสินค้า") },
                scrollBehavior = scrollBehavior,
                navigationIcon = {
                    IconButton(onClick = { }) {
                        Icon(Icons.Default.Menu, contentDescription = "เมนู")
                    }
                }
            )
        },
        modifier = Modifier.nestedScroll(scrollBehavior.nestedScrollConnection)
    ) { paddingValues ->
        LazyColumn(contentPadding = paddingValues) {
            items(50) { index ->
                ListItem(headlineContent = { Text("รายการ ${index + 1}") })
                Divider()
            }
        }
    }
}
```

---

## ขั้นตอนที่ 653: NavigationBar (Bottom Navigation)

```kotlin
import androidx.compose.material.icons.Icons
import androidx.compose.material.icons.filled.*
import androidx.compose.material.icons.outlined.*
import androidx.compose.material3.*
import androidx.compose.runtime.*
import androidx.compose.ui.graphics.vector.ImageVector

data class NavItem(
    val label: String,
    val selectedIcon: ImageVector,
    val unselectedIcon: ImageVector,
    val badgeCount: Int? = null
)

@Composable
fun MainScaffoldWithNavigation() {
    val navItems = listOf(
        NavItem("หน้าแรก", Icons.Filled.Home, Icons.Outlined.Home),
        NavItem("ค้นหา", Icons.Filled.Search, Icons.Outlined.Search),
        NavItem("สั่งซื้อ", Icons.Filled.ShoppingCart, Icons.Outlined.ShoppingCart, badgeCount = 3),
        NavItem("โปรไฟล์", Icons.Filled.Person, Icons.Outlined.Person)
    )
    var selectedIndex by remember { mutableStateOf(0) }

    Scaffold(
        bottomBar = {
            NavigationBar {
                navItems.forEachIndexed { index, item ->
                    NavigationBarItem(
                        selected = selectedIndex == index,
                        onClick = { selectedIndex = index },
                        icon = {
                            if (item.badgeCount != null) {
                                BadgedBox(badge = {
                                    Badge { Text(item.badgeCount.toString()) }
                                }) {
                                    Icon(
                                        if (selectedIndex == index) item.selectedIcon
                                        else item.unselectedIcon,
                                        contentDescription = item.label
                                    )
                                }
                            } else {
                                Icon(
                                    if (selectedIndex == index) item.selectedIcon
                                    else item.unselectedIcon,
                                    contentDescription = item.label
                                )
                            }
                        },
                        label = { Text(item.label) }
                    )
                }
            }
        }
    ) { paddingValues ->
        when (selectedIndex) {
            0 -> HomeScreen()
            1 -> SearchScreen()
            2 -> CartScreen()
            3 -> ProfileScreen()
        }
    }
}
```

---

## ขั้นตอนที่ 654: NavigationDrawer และ NavigationRail

```kotlin
import androidx.compose.foundation.layout.*
import androidx.compose.material3.*
import androidx.compose.runtime.*
import kotlinx.coroutines.launch

@OptIn(ExperimentalMaterial3Api::class)
@Composable
fun NavigationDrawerExample() {
    val drawerState = rememberDrawerState(DrawerValue.Closed)
    val scope = rememberCoroutineScope()
    var selectedItem by remember { mutableStateOf("หน้าแรก") }

    val drawerItems = listOf(
        Triple("หน้าแรก", Icons.Default.Home, "home"),
        Triple("โปรไฟล์", Icons.Default.Person, "profile"),
        Triple("การตั้งค่า", Icons.Default.Settings, "settings"),
        Triple("ช่วยเหลือ", Icons.Default.Help, "help"),
        Triple("ออกจากระบบ", Icons.Default.Logout, "logout")
    )

    ModalNavigationDrawer(
        drawerState = drawerState,
        drawerContent = {
            ModalDrawerSheet {
                // Header
                Box(
                    modifier = Modifier
                        .fillMaxWidth()
                        .padding(16.dp)
                ) {
                    Column {
                        Icon(
                            Icons.Default.AccountCircle,
                            contentDescription = null,
                            modifier = Modifier.size(64.dp)
                        )
                        Text("ผู้ใช้งาน", style = MaterialTheme.typography.titleMedium)
                        Text("user@example.com", style = MaterialTheme.typography.bodySmall)
                    }
                }

                Divider()
                Spacer(Modifier.height(8.dp))

                // Menu Items
                drawerItems.forEach { (label, icon, key) ->
                    NavigationDrawerItem(
                        label = { Text(label) },
                        selected = selectedItem == label,
                        onClick = {
                            selectedItem = label
                            scope.launch { drawerState.close() }
                        },
                        icon = { Icon(icon, contentDescription = label) },
                        modifier = Modifier.padding(NavigationDrawerItemDefaults.ItemPadding)
                    )
                }
            }
        }
    ) {
        Scaffold(
            topBar = {
                TopAppBar(
                    title = { Text(selectedItem) },
                    navigationIcon = {
                        IconButton(onClick = { scope.launch { drawerState.open() } }) {
                            Icon(Icons.Default.Menu, contentDescription = "เมนู")
                        }
                    }
                )
            }
        ) { paddingValues ->
            Box(
                modifier = Modifier
                    .fillMaxSize()
                    .padding(paddingValues),
                contentAlignment = Alignment.Center
            ) {
                Text("$selectedItem Screen")
            }
        }
    }
}
```

---

## ขั้นตอนที่ 655: Dialog, SnackBar, BottomSheet

```kotlin
import androidx.compose.foundation.layout.*
import androidx.compose.material3.*
import androidx.compose.runtime.*
import kotlinx.coroutines.launch

@OptIn(ExperimentalMaterial3Api::class)
@Composable
fun DialogAndSnackbarExamples() {
    val snackbarHostState = remember { SnackbarHostState() }
    val scope = rememberCoroutineScope()
    var showDialog by remember { mutableStateOf(false) }
    var showBottomSheet by remember { mutableStateOf(false) }

    Scaffold(snackbarHost = { SnackbarHost(snackbarHostState) }) { padding ->
        Column(
            modifier = Modifier.padding(padding).padding(16.dp),
            verticalArrangement = Arrangement.spacedBy(8.dp)
        ) {
            // Alert Dialog
            Button(onClick = { showDialog = true }) { Text("แสดง Dialog") }

            // Snackbar
            Button(onClick = {
                scope.launch {
                    val result = snackbarHostState.showSnackbar(
                        message = "ลบรายการแล้ว",
                        actionLabel = "เลิกทำ",
                        duration = SnackbarDuration.Short
                    )
                    if (result == SnackbarResult.ActionPerformed) {
                        // undo action
                    }
                }
            }) { Text("แสดง Snackbar") }

            // Bottom Sheet
            Button(onClick = { showBottomSheet = true }) { Text("แสดง Bottom Sheet") }
        }
    }

    if (showDialog) {
        AlertDialog(
            onDismissRequest = { showDialog = false },
            title = { Text("ยืนยันการลบ") },
            text = { Text("คุณต้องการลบรายการนี้หรือไม่? การกระทำนี้ไม่สามารถย้อนกลับได้") },
            confirmButton = {
                TextButton(onClick = { showDialog = false }) {
                    Text("ลบ", color = MaterialTheme.colorScheme.error)
                }
            },
            dismissButton = {
                TextButton(onClick = { showDialog = false }) { Text("ยกเลิก") }
            },
            icon = { Icon(Icons.Default.Delete, contentDescription = null) }
        )
    }

    if (showBottomSheet) {
        ModalBottomSheet(onDismissRequest = { showBottomSheet = false }) {
            Column(modifier = Modifier.padding(16.dp)) {
                Text("ตัวเลือก", style = MaterialTheme.typography.titleLarge)
                Spacer(Modifier.height(16.dp))
                listOf("แก้ไข", "แชร์", "คัดลอก", "ลบ").forEach { option ->
                    ListItem(
                        headlineContent = { Text(option) },
                        modifier = Modifier.clickable { showBottomSheet = false }
                    )
                }
                Spacer(Modifier.height(32.dp))
            }
        }
    }
}
```

---

*Part 35 จบแล้ว | ก่อนหน้า: [Part 34](../part34/README.md) | ถัดไป: [Part 36](../part36/README.md)*
