# Part 63: Compose Internals & Slot API
## ขั้นตอนที่ 1051-1075

---

## ขั้นตอนที่ 1051: Compose Runtime Internals

```kotlin
// ============================================
// วิธีที่ Compose ทำงาน
// ============================================

/*
 * Compose Phases:
 * 1. Composition - เรียก @Composable functions, สร้าง Composition tree
 * 2. Layout - วัดและวาง nodes (measure → layout)
 * 3. Drawing - วาดบนหน้าจอ
 *
 * Recomposition - เกิดเมื่อ State เปลี่ยน
 * Compose ฉลาดพอที่จะ skip composables ที่ inputs ไม่เปลี่ยน
 *
 * Slot Table - Internal data structure
 * - Compose เก็บ composition state ใน Slot Table
 * - กลุ่ม (Group): เหมือน call stack frame
 * - Slot: ข้อมูลที่เกี่ยวข้องกับแต่ละ group
 */

// Smart Recomposition
@Composable
fun ParentComposable() {
    var count by remember { mutableStateOf(0) }
    
    // RecompositionScope: count เปลี่ยน → ParentComposable recomposes
    // แต่ ChildComposable ที่ params เหมือนเดิม จะ SKIP
    
    ChildA(text = "Static text")  // ← จะไม่ recompose เมื่อ count เปลี่ยน
    ChildB(count = count)          // ← จะ recompose เพราะ count เปลี่ยน
    
    Button(onClick = { count++ }) { Text("Count: $count") }
}

@Composable
fun ChildA(text: String) {
    // State = text string, ถ้า text เหมือนเดิม → skip
    Text(text)
}

@Composable
fun ChildB(count: Int) {
    Text("Count: $count")
}
```

---

## ขั้นตอนที่ 1052: Slot API

```kotlin
// ============================================
// Slot API - เป็น design pattern ใน Compose
// ============================================

// Content Slot คือ @Composable parameter
// ช่วยให้ component flexible สูง

// ตัวอย่าง: Card ที่ customizable ทุกส่วน
@Composable
fun FlexCard(
    modifier: Modifier = Modifier,
    header: (@Composable () -> Unit)? = null,
    content: @Composable () -> Unit,
    footer: (@Composable () -> Unit)? = null,
    actions: (@Composable RowScope.() -> Unit)? = null
) {
    Card(modifier = modifier) {
        Column {
            header?.let { headerContent ->
                Box(modifier = Modifier.padding(16.dp)) {
                    headerContent()
                }
                HorizontalDivider()
            }
            
            Box(modifier = Modifier.padding(16.dp)) {
                content()
            }
            
            footer?.let { footerContent ->
                HorizontalDivider()
                Box(modifier = Modifier.padding(8.dp, 4.dp)) {
                    footerContent()
                }
            }
            
            actions?.let { actionsContent ->
                Row(
                    modifier = Modifier
                        .fillMaxWidth()
                        .padding(8.dp),
                    horizontalArrangement = Arrangement.End,
                    content = actionsContent
                )
            }
        }
    }
}

// ใช้งาน
FlexCard(
    header = {
        Text("ชื่อผลิตภัณฑ์", style = MaterialTheme.typography.titleMedium)
    },
    content = {
        Text("รายละเอียดผลิตภัณฑ์...")
    },
    actions = {
        TextButton(onClick = {}) { Text("ยกเลิก") }
        Button(onClick = {}) { Text("ซื้อเลย") }
    }
)

// ============================================
// Generic Container
// ============================================

@Composable
fun <T> AsyncContent(
    state: AsyncState<T>,
    loadingContent: @Composable () -> Unit = { CircularProgressIndicator() },
    errorContent: @Composable (String) -> Unit = { msg -> Text(msg, color = MaterialTheme.colorScheme.error) },
    emptyContent: @Composable () -> Unit = { Text("ไม่มีข้อมูล") },
    content: @Composable (T) -> Unit
) {
    when (state) {
        is AsyncState.Loading -> loadingContent()
        is AsyncState.Error -> errorContent(state.message)
        is AsyncState.Empty -> emptyContent()
        is AsyncState.Success -> content(state.data)
    }
}

sealed class AsyncState<out T> {
    object Loading : AsyncState<Nothing>()
    object Empty : AsyncState<Nothing>()
    data class Error(val message: String) : AsyncState<Nothing>()
    data class Success<T>(val data: T) : AsyncState<T>()
}

// ใช้งาน
@Composable
fun ProductDetailScreen(viewModel: ProductViewModel = hiltViewModel()) {
    val productState by viewModel.productState.collectAsStateWithLifecycle()
    
    AsyncContent(
        state = productState,
        loadingContent = { LoadingView() },
        errorContent = { msg ->
            ErrorView(
                error = msg,
                onRetry = viewModel::retry
            )
        }
    ) { product ->
        ProductDetail(product = product)
    }
}
```

---

## ขั้นตอนที่ 1053: CompositionLocal

```kotlin
// ============================================
// CompositionLocal - implicit passing ของค่า
// ============================================

// สร้าง CompositionLocal
val LocalAnalytics = staticCompositionLocalOf<AnalyticsTracker> {
    error("No AnalyticsTracker provided")
}

val LocalAppConfig = compositionLocalOf<AppConfig> {
    AppConfig.default()
}

// Provide values
@Composable
fun AppWithAnalytics(
    analytics: AnalyticsTracker,
    config: AppConfig,
    content: @Composable () -> Unit
) {
    CompositionLocalProvider(
        LocalAnalytics provides analytics,
        LocalAppConfig provides config
    ) {
        content()
    }
}

// Consume ได้ทุกที่ใน composable tree
@Composable
fun TrackableButton(
    text: String,
    eventName: String,
    onClick: () -> Unit
) {
    val analytics = LocalAnalytics.current
    val config = LocalAppConfig.current
    
    Button(
        onClick = {
            if (config.analyticsEnabled) {
                analytics.track(AnalyticsEvent(eventName))
            }
            onClick()
        }
    ) { Text(text) }
}

// ============================================
// staticCompositionLocalOf vs compositionLocalOf
// ============================================

// staticCompositionLocalOf:
// - สำหรับค่าที่ไม่ค่อยเปลี่ยน (เช่น theme, config)
// - เมื่อเปลี่ยน → recompose ทุก consumer ใน tree
// - เร็วกว่าในการอ่าน

// compositionLocalOf:
// - สำหรับค่าที่เปลี่ยนบ่อย
// - เมื่อเปลี่ยน → เฉพาะ composables ที่อ่านจะ recompose
// - มี overhead เล็กน้อยในการอ่าน

val LocalThemeColor = compositionLocalOf { Color.Blue }  // เปลี่ยนบ่อย
val LocalActivity = staticCompositionLocalOf<Activity> {  // ไม่ค่อยเปลี่ยน
    error("No Activity")
}
```

---

## ขั้นตอนที่ 1054: Side Effects ขั้นสูง

```kotlin
// ============================================
// LaunchedEffect vs SideEffect vs DisposableEffect
// ============================================

@Composable
fun AnalyticsScreen(screenName: String) {
    val analytics = LocalAnalytics.current
    
    // SideEffect - รันทุก successful composition (ไม่มี cleanup)
    SideEffect {
        // ใช้สำหรับ sync ค่า non-compose systems
        analytics.setCurrentScreen(screenName)
    }
    
    // LaunchedEffect - suspend function, รีสตาร์ทเมื่อ key เปลี่ยน
    LaunchedEffect(screenName) {
        analytics.trackScreenView(screenName)
        delay(5000)
        analytics.trackEngagement(screenName, duration = 5000)
    }
    
    // DisposableEffect - มี cleanup
    DisposableEffect(Unit) {
        analytics.onScreenEnter(screenName)
        onDispose {
            analytics.onScreenExit(screenName)
        }
    }
}

// ============================================
// rememberCoroutineScope
// ============================================

@Composable
fun ScrollableContent(items: List<String>) {
    val listState = rememberLazyListState()
    val coroutineScope = rememberCoroutineScope()
    
    Box {
        LazyColumn(state = listState) {
            items(items) { item -> Text(item) }
        }
        
        // FloatingActionButton ที่ scroll กลับบนสุด
        AnimatedVisibility(
            visible = listState.firstVisibleItemIndex > 5,
            modifier = Modifier.align(Alignment.BottomEnd).padding(16.dp)
        ) {
            FloatingActionButton(
                onClick = {
                    coroutineScope.launch {
                        listState.animateScrollToItem(0)
                    }
                }
            ) {
                Icon(Icons.Default.KeyboardArrowUp, contentDescription = "Scroll to top")
            }
        }
    }
}

// ============================================
// produceState - convert non-compose state to Compose state
// ============================================

@Composable
fun LocationDisplay() {
    val locationState = produceState<Location?>(initialValue = null) {
        val provider = LocationProvider()
        
        provider.startUpdates { location ->
            value = location
        }
        
        awaitDispose {
            provider.stopUpdates()
        }
    }
    
    Text(locationState.value?.let { "${it.lat}, ${it.lng}" } ?: "กำลังระบุตำแหน่ง...")
}
```

---

## แบบฝึกหัด Part 63

```kotlin
// แบบฝึกหัด: สร้าง Custom Scaffold ด้วย Slot API

// สร้าง ResponsiveScaffold ที่:
// - บน Phone: แสดง BottomNavigation
// - บน Tablet: แสดง NavigationRail หรือ NavigationDrawer

@Composable
fun ResponsiveScaffold(
    currentDestination: String,
    destinations: List<NavDestination>,
    onDestinationChange: (String) -> Unit,
    topBar: @Composable () -> Unit = {},
    content: @Composable (PaddingValues) -> Unit
) {
    val windowSizeClass = calculateWindowSizeClass()
    
    when (windowSizeClass.widthSizeClass) {
        WindowWidthSizeClass.Compact -> {
            // Phone: BottomNavigation
            Scaffold(
                topBar = topBar,
                bottomBar = {
                    NavigationBar {
                        destinations.forEach { dest ->
                            NavigationBarItem(
                                selected = currentDestination == dest.route,
                                onClick = { onDestinationChange(dest.route) },
                                icon = { Icon(dest.icon, null) },
                                label = { Text(dest.label) }
                            )
                        }
                    }
                },
                content = content
            )
        }
        
        else -> {
            // Tablet/Desktop: NavigationRail
            Row {
                NavigationRail {
                    destinations.forEach { dest ->
                        NavigationRailItem(
                            selected = currentDestination == dest.route,
                            onClick = { onDestinationChange(dest.route) },
                            icon = { Icon(dest.icon, null) },
                            label = { Text(dest.label) }
                        )
                    }
                }
                
                Scaffold(topBar = topBar, content = content)
            }
        }
    }
}

data class NavDestination(
    val route: String,
    val label: String,
    val icon: ImageVector
)
```

---

*Part 63 จบแล้ว | ก่อนหน้า: [Part 62](../part62/README.md) | ถัดไป: [Part 64](../part64/README.md)*
