# Part 40: Compose - Side Effects
## ขั้นตอนที่ 776-800

---

## ขั้นตอนที่ 776: LaunchedEffect - รัน Coroutine ใน Composable

LaunchedEffect รัน coroutine เมื่อ key เปลี่ยน และยกเลิกอัตโนมัติเมื่อออกจาก composition

```kotlin
import androidx.compose.runtime.*
import kotlinx.coroutines.delay

@Composable
fun LaunchedEffectExamples() {
    var userId by remember { mutableStateOf(1) }
    var userData by remember { mutableStateOf<String?>(null) }
    var isLoading by remember { mutableStateOf(false) }

    // LaunchedEffect(key) - รันเมื่อ key เปลี่ยน
    LaunchedEffect(userId) {
        isLoading = true
        userData = null
        try {
            delay(1000) // จำลอง API call
            userData = "ข้อมูลของผู้ใช้ #$userId"
        } finally {
            isLoading = false
        }
    }

    Column(modifier = Modifier.padding(16.dp)) {
        Text("ผู้ใช้: #$userId")

        Row(horizontalArrangement = Arrangement.spacedBy(8.dp)) {
            Button(onClick = { if (userId > 1) userId-- }) { Text("ก่อนหน้า") }
            Button(onClick = { userId++ }) { Text("ถัดไป") }
        }

        if (isLoading) {
            CircularProgressIndicator()
        } else {
            Text(userData ?: "ไม่มีข้อมูล")
        }
    }
}

// LaunchedEffect(Unit) - รันครั้งเดียวตอน Composable เข้าหน้าจอ
@Composable
fun OneTimeEffectExample() {
    var message by remember { mutableStateOf("กำลังโหลด...") }

    LaunchedEffect(Unit) {
        // รันครั้งเดียวตอนแสดงครั้งแรก
        delay(2000)
        message = "โหลดสำเร็จ!"
    }

    Text(message)
}

// LaunchedEffect กับ Flow
@Composable
fun CollectFlowInLaunchedEffect(viewModel: MyViewModel) {
    val snackbarHostState = remember { SnackbarHostState() }

    // collect flow ใน LaunchedEffect
    LaunchedEffect(viewModel) {
        viewModel.events.collect { event ->
            when (event) {
                is UiEvent.ShowSnackbar -> snackbarHostState.showSnackbar(event.message)
                is UiEvent.Navigate -> { /* navigate */ }
            }
        }
    }

    Scaffold(snackbarHost = { SnackbarHost(snackbarHostState) }) { /* content */ }
}
```

---

## ขั้นตอนที่ 777: DisposableEffect - Cleanup เมื่อออกจาก Composition

DisposableEffect ใช้เมื่อต้องการ register/unregister listener เช่น lifecycle, sensor

```kotlin
import androidx.compose.runtime.*
import androidx.lifecycle.Lifecycle
import androidx.lifecycle.LifecycleEventObserver

@Composable
fun LifecycleObserverEffect(
    lifecycleOwner: LifecycleOwner = LocalLifecycleOwner.current
) {
    DisposableEffect(lifecycleOwner) {
        val observer = LifecycleEventObserver { _, event ->
            when (event) {
                Lifecycle.Event.ON_RESUME -> println("Screen ON_RESUME")
                Lifecycle.Event.ON_PAUSE -> println("Screen ON_PAUSE")
                Lifecycle.Event.ON_STOP -> println("Screen ON_STOP")
                else -> {}
            }
        }

        lifecycleOwner.lifecycle.addObserver(observer)

        // onDispose - รันตอน Composable ออกจากหน้าจอ
        onDispose {
            lifecycleOwner.lifecycle.removeObserver(observer)
            println("Observer removed!")
        }
    }
}

// DisposableEffect กับ BroadcastReceiver
@Composable
fun NetworkStateEffect(onNetworkChange: (Boolean) -> Unit) {
    val context = LocalContext.current

    DisposableEffect(context) {
        val receiver = object : BroadcastReceiver() {
            override fun onReceive(ctx: Context, intent: Intent) {
                val cm = ctx.getSystemService(Context.CONNECTIVITY_SERVICE) as ConnectivityManager
                val isConnected = cm.activeNetworkInfo?.isConnected == true
                onNetworkChange(isConnected)
            }
        }

        val filter = IntentFilter(ConnectivityManager.CONNECTIVITY_ACTION)
        context.registerReceiver(receiver, filter)

        onDispose {
            context.unregisterReceiver(receiver)
        }
    }
}

// DisposableEffect กับ MediaPlayer
@Composable
fun MediaPlayerEffect(url: String) {
    var isPlaying by remember { mutableStateOf(false) }

    DisposableEffect(url) {
        val player = MediaPlayer().apply {
            setDataSource(url)
            prepareAsync()
            setOnPreparedListener { start(); isPlaying = true }
        }

        onDispose {
            player.stop()
            player.release()
        }
    }
}
```

---

## ขั้นตอนที่ 778: SideEffect และ rememberUpdatedState

```kotlin
import androidx.compose.runtime.*

// SideEffect - รันทุกครั้งที่ recompose (ไม่มี cleanup)
// ใช้สำหรับส่งค่าไปยัง non-Compose code
@Composable
fun SideEffectExample(analyticsTracker: AnalyticsTracker) {
    var screenName by remember { mutableStateOf("home") }

    // SideEffect รันทุกครั้งที่ recompose สำเร็จ
    SideEffect {
        analyticsTracker.setCurrentScreen(screenName)
    }
    // ...
}

// rememberUpdatedState - capture ค่าล่าสุดใน effect
@Composable
fun TimerEffect(onTimeout: () -> Unit) {
    // ถ้า onTimeout เปลี่ยน ไม่ต้องรัน LaunchedEffect ใหม่
    val currentOnTimeout by rememberUpdatedState(onTimeout)

    LaunchedEffect(Unit) { // key = Unit จึงรันครั้งเดียว
        delay(5000)
        currentOnTimeout() // ใช้ค่าล่าสุดเสมอ
    }
}

// ตัวอย่างปัญหาที่ rememberUpdatedState แก้ได้
@Composable
fun CorrectWayToPassCallback() {
    var count by remember { mutableStateOf(0) }

    // ถ้าไม่ใช้ rememberUpdatedState:
    // LaunchedEffect จะ capture onTimeout ณ ตอน compose ครั้งแรก
    // ถ้า count เปลี่ยน closure จะยังเห็น count เก่า

    val currentCount by rememberUpdatedState(count)

    LaunchedEffect(Unit) {
        delay(3000)
        println("Count after 3 seconds: $currentCount") // ได้ค่าล่าสุด
    }

    Button(onClick = { count++ }) {
        Text("Count: $count")
    }
}
```

---

## ขั้นตอนที่ 779: rememberCoroutineScope - เรียก suspend จาก Event Handler

```kotlin
import androidx.compose.runtime.rememberCoroutineScope
import kotlinx.coroutines.launch

@Composable
fun CoroutineScopeExample() {
    // scope ที่ผูกกับ Composable lifecycle
    val scope = rememberCoroutineScope()
    val snackbarHostState = remember { SnackbarHostState() }
    var isLoading by remember { mutableStateOf(false) }

    Scaffold(snackbarHost = { SnackbarHost(snackbarHostState) }) { padding ->
        Column(modifier = Modifier.padding(padding).padding(16.dp)) {
            Button(
                onClick = {
                    // ใช้ scope.launch ใน event handler
                    scope.launch {
                        isLoading = true
                        try {
                            delay(2000)
                            snackbarHostState.showSnackbar("บันทึกสำเร็จ!")
                        } catch (e: Exception) {
                            snackbarHostState.showSnackbar("เกิดข้อผิดพลาด: ${e.message}")
                        } finally {
                            isLoading = false
                        }
                    }
                },
                enabled = !isLoading
            ) {
                if (isLoading) {
                    CircularProgressIndicator(modifier = Modifier.size(20.dp))
                } else {
                    Text("บันทึก")
                }
            }
        }
    }
}

// produceState - แปลง non-Compose state เป็น State
@Composable
fun NetworkImageUrl(url: String): State<Bitmap?> {
    return produceState<Bitmap?>(initialValue = null, url) {
        // รันใน coroutine
        value = withContext(Dispatchers.IO) {
            try {
                val stream = java.net.URL(url).openStream()
                BitmapFactory.decodeStream(stream)
            } catch (e: Exception) {
                null
            }
        }
    }
}

// derivedStateOf - compute state จาก state อื่น (cached)
@Composable
fun DerivedStateExample() {
    var text by remember { mutableStateOf("") }

    // derivedStateOf - recompute เฉพาะเมื่อ input เปลี่ยน
    val isValid by remember {
        derivedStateOf { text.length >= 3 && text.all { it.isLetterOrDigit() } }
    }

    Column {
        OutlinedTextField(value = text, onValueChange = { text = it })
        Text(
            text = if (isValid) "✓ ถูกต้อง" else "✗ ไม่ถูกต้อง",
            color = if (isValid) Color.Green else Color.Red
        )
    }
}
```

---

## ขั้นตอนที่ 780: snapshotFlow - แปลง State เป็น Flow

```kotlin
import androidx.compose.runtime.snapshotFlow
import kotlinx.coroutines.flow.distinctUntilChanged
import kotlinx.coroutines.flow.filter

@Composable
fun SnapshotFlowExample() {
    val listState = rememberLazyListState()
    var showScrollToTop by remember { mutableStateOf(false) }

    // snapshotFlow แปลง Compose State เป็น Flow
    LaunchedEffect(listState) {
        snapshotFlow { listState.firstVisibleItemIndex }
            .distinctUntilChanged()
            .collect { index ->
                showScrollToTop = index > 5
            }
    }

    Box {
        LazyColumn(state = listState) {
            items(100) { index ->
                ListItem(headlineContent = { Text("รายการ ${index + 1}") })
                Divider()
            }
        }

        if (showScrollToTop) {
            val scope = rememberCoroutineScope()
            FloatingActionButton(
                onClick = { scope.launch { listState.animateScrollToItem(0) } },
                modifier = Modifier
                    .align(Alignment.BottomEnd)
                    .padding(16.dp)
            ) {
                Icon(Icons.Default.KeyboardArrowUp, "เลื่อนขึ้น")
            }
        }
    }
}

// ตัวอย่าง Infinite scroll
@Composable
fun InfiniteScrollExample(viewModel: InfiniteScrollViewModel) {
    val listState = rememberLazyListState()
    val items by viewModel.items.collectAsStateWithLifecycle()

    LaunchedEffect(listState) {
        snapshotFlow {
            val layoutInfo = listState.layoutInfo
            val totalItems = layoutInfo.totalItemsCount
            val lastVisibleIndex = layoutInfo.visibleItemsInfo.lastOrNull()?.index ?: 0
            lastVisibleIndex >= totalItems - 5 // ถึง 5 รายการก่อนสุดท้าย
        }
            .distinctUntilChanged()
            .filter { it }
            .collect {
                viewModel.loadMore()
            }
    }

    LazyColumn(state = listState) {
        items(items) { item ->
            Text(item, modifier = Modifier.padding(16.dp))
            Divider()
        }
        if (viewModel.isLoading) {
            item {
                Box(Modifier.fillMaxWidth().padding(8.dp), Alignment.Center) {
                    CircularProgressIndicator()
                }
            }
        }
    }
}
```

---

*Part 40 จบแล้ว | ก่อนหน้า: [Part 39](../part39/README.md) | ถัดไป: [Part 41](../part41/README.md)*
