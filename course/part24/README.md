# Part 24: Navigation ใน Jetpack Compose
## ขั้นตอนที่ 576-600

---

## ขั้นตอนที่ 576: Navigation Compose Setup

```kotlin
// build.gradle.kts - dependencies
// implementation("androidx.navigation:navigation-compose:2.8.x")
// implementation("androidx.hilt:hilt-navigation-compose:1.2.x")

// ============================================
// NavHost และ NavController
// ============================================

@Composable
fun AppNavigation() {
    val navController = rememberNavController()
    
    NavHost(
        navController = navController,
        startDestination = "home"
    ) {
        composable("home") {
            HomeScreen(navController = navController)
        }
        
        composable("profile") {
            ProfileScreen(navController = navController)
        }
        
        composable("detail/{userId}") { backStackEntry ->
            val userId = backStackEntry.arguments?.getString("userId")
            DetailScreen(userId = userId, navController = navController)
        }
    }
}

// ใช้ NavController
@Composable
fun HomeScreen(navController: NavController) {
    Column {
        Button(onClick = { navController.navigate("profile") }) {
            Text("ไปหน้า Profile")
        }
        
        Button(onClick = { navController.navigate("detail/123") }) {
            Text("ดู User 123")
        }
    }
}
```

---

## ขั้นตอนที่ 577: Type-safe Navigation ด้วย Sealed Class

```kotlin
// ============================================
// Type-safe Routes
// ============================================

sealed class Screen(val route: String) {
    object Home : Screen("home")
    object Profile : Screen("profile/{userId}") {
        fun createRoute(userId: Long) = "profile/$userId"
    }
    object Settings : Screen("settings")
    object PostDetail : Screen("post/{postId}?source={source}") {
        fun createRoute(postId: Long, source: String = "feed") =
            "post/$postId?source=$source"
    }
}

// NavHost ที่ใช้ type-safe routes
@Composable
fun TypeSafeNavHost(navController: NavHostController) {
    NavHost(
        navController = navController,
        startDestination = Screen.Home.route
    ) {
        composable(Screen.Home.route) {
            HomeScreen(navController)
        }
        
        composable(
            route = Screen.Profile.route,
            arguments = listOf(
                navArgument("userId") {
                    type = NavType.LongType
                }
            )
        ) { entry ->
            val userId = entry.arguments!!.getLong("userId")
            ProfileScreen(userId = userId, navController = navController)
        }
        
        composable(
            route = Screen.PostDetail.route,
            arguments = listOf(
                navArgument("postId") { type = NavType.LongType },
                navArgument("source") {
                    type = NavType.StringType
                    defaultValue = "feed"
                    nullable = false
                }
            )
        ) { entry ->
            val postId = entry.arguments!!.getLong("postId")
            val source = entry.arguments!!.getString("source") ?: "feed"
            PostDetailScreen(postId = postId, source = source)
        }
    }
}

// Navigation extension functions
fun NavController.navigateToProfile(userId: Long) {
    navigate(Screen.Profile.createRoute(userId))
}

fun NavController.navigateToPostDetail(postId: Long, source: String = "feed") {
    navigate(Screen.PostDetail.createRoute(postId, source))
}

fun NavController.popBackStackTo(route: String, inclusive: Boolean = false) {
    popBackStack(route, inclusive)
}
```

---

## ขั้นตอนที่ 578: Bottom Navigation Bar

```kotlin
// ============================================
// Bottom Navigation ที่สมบูรณ์
// ============================================

sealed class BottomNavItem(
    val route: String,
    val title: String,
    val icon: ImageVector,
    val selectedIcon: ImageVector = icon
) {
    object Home : BottomNavItem(
        route = "home",
        title = "หน้าแรก",
        icon = Icons.Outlined.Home,
        selectedIcon = Icons.Filled.Home
    )
    object Search : BottomNavItem(
        route = "search",
        title = "ค้นหา",
        icon = Icons.Outlined.Search,
        selectedIcon = Icons.Filled.Search
    )
    object Notifications : BottomNavItem(
        route = "notifications",
        title = "แจ้งเตือน",
        icon = Icons.Outlined.Notifications,
        selectedIcon = Icons.Filled.Notifications
    )
    object Profile : BottomNavItem(
        route = "profile",
        title = "โปรไฟล์",
        icon = Icons.Outlined.Person,
        selectedIcon = Icons.Filled.Person
    )
}

@Composable
fun MainApp() {
    val navController = rememberNavController()
    
    Scaffold(
        bottomBar = {
            MainBottomBar(navController = navController)
        }
    ) { paddingValues ->
        NavHost(
            navController = navController,
            startDestination = BottomNavItem.Home.route,
            modifier = Modifier.padding(paddingValues)
        ) {
            composable(BottomNavItem.Home.route) { HomeScreen(navController) }
            composable(BottomNavItem.Search.route) { SearchScreen(navController) }
            composable(BottomNavItem.Notifications.route) { NotificationsScreen() }
            composable(BottomNavItem.Profile.route) { ProfileScreen(navController) }
            
            // Nested routes
            composable("post_detail/{postId}") { entry ->
                val postId = entry.arguments?.getString("postId")?.toLong() ?: return@composable
                PostDetailScreen(postId = postId, navController = navController)
            }
        }
    }
}

@Composable
fun MainBottomBar(navController: NavController) {
    val items = listOf(
        BottomNavItem.Home,
        BottomNavItem.Search,
        BottomNavItem.Notifications,
        BottomNavItem.Profile
    )
    
    val navBackStackEntry by navController.currentBackStackEntryAsState()
    val currentRoute = navBackStackEntry?.destination?.route
    
    // แสดง bottom bar เฉพาะ top-level routes
    val showBottomBar = items.any { it.route == currentRoute }
    
    AnimatedVisibility(
        visible = showBottomBar,
        enter = slideInVertically(initialOffsetY = { it }),
        exit = slideOutVertically(targetOffsetY = { it })
    ) {
        NavigationBar {
            items.forEach { item ->
                val isSelected = currentRoute == item.route
                
                NavigationBarItem(
                    icon = {
                        Icon(
                            imageVector = if (isSelected) item.selectedIcon else item.icon,
                            contentDescription = item.title
                        )
                    },
                    label = { Text(item.title) },
                    selected = isSelected,
                    onClick = {
                        navController.navigate(item.route) {
                            // Pop up to start destination เพื่อหลีกเลี่ยง back stack ซ้ำ
                            popUpTo(navController.graph.findStartDestination().id) {
                                saveState = true
                            }
                            launchSingleTop = true
                            restoreState = true
                        }
                    }
                )
            }
        }
    }
}
```

---

## ขั้นตอนที่ 579: Navigation Drawer

```kotlin
@Composable
fun DrawerNavigation() {
    val drawerState = rememberDrawerState(DrawerValue.Closed)
    val scope = rememberCoroutineScope()
    val navController = rememberNavController()
    
    ModalNavigationDrawer(
        drawerState = drawerState,
        drawerContent = {
            ModalDrawerSheet {
                DrawerHeader()
                DrawerBody(
                    navController = navController,
                    onItemClick = {
                        scope.launch { drawerState.close() }
                    }
                )
            }
        }
    ) {
        Scaffold(
            topBar = {
                TopAppBar(
                    title = { Text("แอปพลิเคชัน") },
                    navigationIcon = {
                        IconButton(
                            onClick = { scope.launch { drawerState.open() } }
                        ) {
                            Icon(Icons.Default.Menu, contentDescription = "Menu")
                        }
                    }
                )
            }
        ) { padding ->
            NavHost(
                navController = navController,
                startDestination = "home",
                modifier = Modifier.padding(padding)
            ) {
                composable("home") { HomeScreen(navController) }
                composable("settings") { SettingsScreen() }
                composable("about") { AboutScreen() }
            }
        }
    }
}

@Composable
fun DrawerHeader() {
    Box(
        modifier = Modifier
            .fillMaxWidth()
            .height(200.dp)
            .background(MaterialTheme.colorScheme.primaryContainer),
        contentAlignment = Alignment.BottomStart
    ) {
        Column(modifier = Modifier.padding(16.dp)) {
            Icon(
                Icons.Default.AccountCircle,
                contentDescription = null,
                modifier = Modifier.size(64.dp),
                tint = MaterialTheme.colorScheme.onPrimaryContainer
            )
            Spacer(modifier = Modifier.height(8.dp))
            Text(
                "ชื่อผู้ใช้",
                style = MaterialTheme.typography.titleLarge,
                color = MaterialTheme.colorScheme.onPrimaryContainer
            )
            Text(
                "user@example.com",
                style = MaterialTheme.typography.bodyMedium,
                color = MaterialTheme.colorScheme.onPrimaryContainer
            )
        }
    }
}

data class DrawerMenuItem(
    val route: String,
    val title: String,
    val icon: ImageVector
)

@Composable
fun DrawerBody(navController: NavController, onItemClick: () -> Unit) {
    val items = listOf(
        DrawerMenuItem("home", "หน้าแรก", Icons.Default.Home),
        DrawerMenuItem("settings", "ตั้งค่า", Icons.Default.Settings),
        DrawerMenuItem("about", "เกี่ยวกับ", Icons.Default.Info)
    )
    
    val currentRoute = navController.currentBackStackEntry?.destination?.route
    
    items.forEach { item ->
        NavigationDrawerItem(
            icon = { Icon(item.icon, contentDescription = null) },
            label = { Text(item.title) },
            selected = currentRoute == item.route,
            onClick = {
                navController.navigate(item.route) {
                    launchSingleTop = true
                    restoreState = true
                }
                onItemClick()
            },
            modifier = Modifier.padding(NavigationDrawerItemDefaults.ItemPadding)
        )
    }
}
```

---

## ขั้นตอนที่ 580: Deep Links

```kotlin
// AndroidManifest.xml
// <activity android:name=".MainActivity">
//     <intent-filter android:autoVerify="true">
//         <action android:name="android.intent.action.VIEW" />
//         <category android:name="android.intent.category.DEFAULT" />
//         <category android:name="android.intent.category.BROWSABLE" />
//         <data android:scheme="https" android:host="example.com" />
//     </intent-filter>
// </activity>

@Composable
fun DeepLinkNavHost() {
    val navController = rememberNavController()
    
    NavHost(
        navController = navController,
        startDestination = "home"
    ) {
        composable("home") { HomeScreen(navController) }
        
        composable(
            route = "user/{userId}",
            arguments = listOf(
                navArgument("userId") { type = NavType.LongType }
            ),
            deepLinks = listOf(
                navDeepLink {
                    uriPattern = "https://example.com/user/{userId}"
                },
                navDeepLink {
                    uriPattern = "myapp://user/{userId}"
                    action = Intent.ACTION_VIEW
                }
            )
        ) { entry ->
            val userId = entry.arguments!!.getLong("userId")
            UserProfileScreen(userId = userId)
        }
        
        composable(
            route = "post/{postId}",
            deepLinks = listOf(
                navDeepLink {
                    uriPattern = "https://example.com/posts/{postId}"
                }
            )
        ) { entry ->
            val postId = entry.arguments?.getString("postId")
            PostDetailScreen(postId = postId?.toLong() ?: 0)
        }
    }
}
```

---

## ขั้นตอนที่ 581: Passing Data Back

```kotlin
// ============================================
// ส่งข้อมูลกลับ ด้วย SavedStateHandle
// ============================================

// Screen ที่รับผลลัพธ์
@Composable
fun ParentScreen(navController: NavController) {
    // รับผลลัพธ์จาก child screen
    val result = navController
        .currentBackStackEntry
        ?.savedStateHandle
        ?.getStateFlow<String?>("result", null)
        ?.collectAsStateWithLifecycle()
    
    Column {
        Text("ผลลัพธ์: ${result?.value ?: "ยังไม่มี"}")
        
        Button(onClick = { navController.navigate("picker") }) {
            Text("เลือก")
        }
    }
}

// Screen ที่ส่งผลลัพธ์กลับ
@Composable
fun PickerScreen(navController: NavController) {
    val options = listOf("Option A", "Option B", "Option C")
    
    Column {
        options.forEach { option ->
            ListItem(
                headlineContent = { Text(option) },
                modifier = Modifier.clickable {
                    // ส่งผลลัพธ์กลับ
                    navController.previousBackStackEntry
                        ?.savedStateHandle
                        ?.set("result", option)
                    navController.popBackStack()
                }
            )
        }
    }
}

// ============================================
// Navigation ผ่าน ViewModel (Pattern ที่ดีกว่า)
// ============================================

@HiltViewModel
class NavigationViewModel @Inject constructor() : ViewModel() {
    private val _navigationEvent = Channel<NavigationEvent>()
    val navigationEvent = _navigationEvent.receiveAsFlow()
    
    fun navigateToDetail(id: Long) {
        viewModelScope.launch {
            _navigationEvent.send(NavigationEvent.ToDetail(id))
        }
    }
    
    fun navigateBack() {
        viewModelScope.launch {
            _navigationEvent.send(NavigationEvent.Back)
        }
    }
}

sealed class NavigationEvent {
    data class ToDetail(val id: Long) : NavigationEvent()
    object Back : NavigationEvent()
}

@Composable
fun ScreenWithNavViewModel(navController: NavController) {
    val viewModel: NavigationViewModel = hiltViewModel()
    
    LaunchedEffect(Unit) {
        viewModel.navigationEvent.collect { event ->
            when (event) {
                is NavigationEvent.ToDetail -> navController.navigate("detail/${event.id}")
                NavigationEvent.Back -> navController.popBackStack()
            }
        }
    }
    
    // UI
    Button(onClick = { viewModel.navigateToDetail(42) }) {
        Text("ดูรายละเอียด")
    }
}
```

---

## ขั้นตอนที่ 582: Nested Navigation Graphs

```kotlin
@Composable
fun AppWithNestedNav() {
    val navController = rememberNavController()
    
    NavHost(navController = navController, startDestination = "auth") {
        // Auth flow
        navigation(startDestination = "login", route = "auth") {
            composable("login") { LoginScreen(navController) }
            composable("register") { RegisterScreen(navController) }
            composable("forgot_password") { ForgotPasswordScreen(navController) }
        }
        
        // Main flow
        navigation(startDestination = "home", route = "main") {
            composable("home") { HomeScreen(navController) }
            composable("search") { SearchScreen(navController) }
            composable("post/{id}") { entry ->
                val id = entry.arguments!!.getString("id")!!.toLong()
                PostScreen(postId = id, navController = navController)
            }
        }
        
        // Settings flow
        navigation(startDestination = "settings_home", route = "settings") {
            composable("settings_home") { SettingsScreen(navController) }
            composable("settings_notifications") { NotificationSettingsScreen() }
            composable("settings_privacy") { PrivacySettingsScreen() }
        }
    }
}

// Navigate to nested graph
fun NavController.navigateToMain() = navigate("main") {
    popUpTo("auth") { inclusive = true }
}

fun NavController.navigateToAuth() = navigate("auth") {
    popUpTo(0) { inclusive = true }
}
```

---

## แบบฝึกหัด Part 24

```kotlin
// แบบฝึกหัด: สร้าง Shopping App Navigation

// Routes
sealed class ShopScreen(val route: String) {
    object ProductList : ShopScreen("products")
    object ProductDetail : ShopScreen("product/{id}") {
        fun createRoute(id: Long) = "product/$id"
    }
    object Cart : ShopScreen("cart")
    object Checkout : ShopScreen("checkout")
    object OrderConfirmation : ShopScreen("order/{orderId}") {
        fun createRoute(orderId: String) = "order/$orderId"
    }
}

// TODO: สร้าง NavHost ที่รองรับทุก routes
// TODO: สร้าง Bottom navigation ที่มี ProductList และ Cart
// TODO: ส่ง product id ไปยัง ProductDetail
// TODO: หลัง checkout สำเร็จ navigate ไป OrderConfirmation
//       และ clear cart และ checkout จาก back stack
```

---

*Part 24 จบแล้ว | ก่อนหน้า: [Part 23](../part23/README.md) | ถัดไป: [Part 25](../part25/README.md)*
