# Part 38: Compose - Navigation พื้นฐาน
## ขั้นตอนที่ 726-750

---

## ขั้นตอนที่ 726: ติดตั้งและตั้งค่า Navigation Compose

Navigation Compose ช่วยจัดการการเดินทางระหว่าง Screen ใน Compose app

```kotlin
// build.gradle.kts (app)
dependencies {
    implementation("androidx.navigation:navigation-compose:2.7.5")
}

// ต้องมี sealed class หรือ object สำหรับ Route
sealed class Screen(val route: String) {
    object Home : Screen("home")
    object Detail : Screen("detail/{itemId}") {
        fun createRoute(itemId: Int) = "detail/$itemId"
    }
    object Profile : Screen("profile")
    object Settings : Screen("settings")
    object Search : Screen("search?query={query}") {
        fun createRoute(query: String = "") = "search?query=$query"
    }
}
```

---

## ขั้นตอนที่ 727: NavHost และ NavController พื้นฐาน

```kotlin
import androidx.compose.runtime.Composable
import androidx.navigation.NavHostController
import androidx.navigation.NavType
import androidx.navigation.compose.*
import androidx.navigation.navArgument

@Composable
fun AppNavigation() {
    // สร้าง NavController
    val navController = rememberNavController()

    // NavHost - กำหนด navigation graph
    NavHost(
        navController = navController,
        startDestination = Screen.Home.route
    ) {
        // กำหนดแต่ละ Screen
        composable(route = Screen.Home.route) {
            HomeScreen(
                onNavigateToDetail = { itemId ->
                    navController.navigate(Screen.Detail.createRoute(itemId))
                },
                onNavigateToProfile = {
                    navController.navigate(Screen.Profile.route)
                }
            )
        }

        // Screen พร้อม argument
        composable(
            route = Screen.Detail.route,
            arguments = listOf(
                navArgument("itemId") { type = NavType.IntType }
            )
        ) { backStackEntry ->
            val itemId = backStackEntry.arguments?.getInt("itemId") ?: 0
            DetailScreen(
                itemId = itemId,
                onBack = { navController.popBackStack() }
            )
        }

        composable(route = Screen.Profile.route) {
            ProfileScreen(
                onBack = { navController.popBackStack() }
            )
        }

        // Screen พร้อม optional argument
        composable(
            route = Screen.Search.route,
            arguments = listOf(
                navArgument("query") {
                    type = NavType.StringType
                    defaultValue = ""
                }
            )
        ) { backStackEntry ->
            val query = backStackEntry.arguments?.getString("query") ?: ""
            SearchScreen(initialQuery = query)
        }
    }
}
```

---

## ขั้นตอนที่ 728: Navigation พร้อม Bottom Navigation

```kotlin
import androidx.compose.material.icons.Icons
import androidx.compose.material.icons.filled.*
import androidx.compose.material3.*
import androidx.compose.runtime.*
import androidx.compose.ui.graphics.vector.ImageVector
import androidx.navigation.NavGraph.Companion.findStartDestination
import androidx.navigation.compose.*

data class BottomNavItem(
    val route: String,
    val title: String,
    val icon: ImageVector
)

val bottomNavItems = listOf(
    BottomNavItem(Screen.Home.route, "หน้าแรก", Icons.Default.Home),
    BottomNavItem(Screen.Search.route, "ค้นหา", Icons.Default.Search),
    BottomNavItem(Screen.Profile.route, "โปรไฟล์", Icons.Default.Person),
    BottomNavItem(Screen.Settings.route, "ตั้งค่า", Icons.Default.Settings)
)

@Composable
fun MainScreen() {
    val navController = rememberNavController()
    val navBackStackEntry by navController.currentBackStackEntryAsState()
    val currentRoute = navBackStackEntry?.destination?.route

    Scaffold(
        bottomBar = {
            // แสดง bottom nav เฉพาะ top-level screens
            val showBottomBar = bottomNavItems.any { it.route == currentRoute }
            if (showBottomBar) {
                NavigationBar {
                    bottomNavItems.forEach { item ->
                        NavigationBarItem(
                            selected = currentRoute == item.route,
                            onClick = {
                                navController.navigate(item.route) {
                                    // กลับไป start destination แทนการ stack
                                    popUpTo(navController.graph.findStartDestination().id) {
                                        saveState = true
                                    }
                                    // ไม่ copy เมื่อคลิกซ้ำ
                                    launchSingleTop = true
                                    // restore state เมื่อกลับมา
                                    restoreState = true
                                }
                            },
                            icon = { Icon(item.icon, contentDescription = item.title) },
                            label = { Text(item.title) }
                        )
                    }
                }
            }
        }
    ) { paddingValues ->
        NavHost(
            navController = navController,
            startDestination = Screen.Home.route,
            modifier = Modifier.padding(paddingValues)
        ) {
            composable(Screen.Home.route) {
                HomeScreen(
                    onNavigateToDetail = { id ->
                        navController.navigate(Screen.Detail.createRoute(id))
                    }
                )
            }
            composable(Screen.Search.route) { SearchScreen() }
            composable(Screen.Profile.route) { ProfileScreen() }
            composable(Screen.Settings.route) { SettingsScreen() }
            composable(
                Screen.Detail.route,
                arguments = listOf(navArgument("itemId") { type = NavType.IntType })
            ) { backStackEntry ->
                DetailScreen(
                    itemId = backStackEntry.arguments?.getInt("itemId") ?: 0,
                    onBack = { navController.popBackStack() }
                )
            }
        }
    }
}
```

---

## ขั้นตอนที่ 729: ส่งข้อมูลระหว่าง Screen

```kotlin
import androidx.compose.foundation.layout.*
import androidx.compose.material3.*
import androidx.compose.runtime.*
import androidx.compose.ui.Modifier
import androidx.compose.ui.unit.dp
import androidx.navigation.NavController

// วิธีที่ 1: ส่งผ่าน route argument (ข้อมูลเล็กน้อย)
@Composable
fun HomeScreen(
    navController: NavController,
    onNavigateToDetail: (Int) -> Unit
) {
    Column(modifier = Modifier.padding(16.dp)) {
        Button(onClick = { onNavigateToDetail(42) }) {
            Text("ดูรายละเอียด Item #42")
        }

        // ส่ง String
        Button(onClick = {
            navController.navigate(Screen.Search.createRoute("kotlin"))
        }) {
            Text("ค้นหา kotlin")
        }
    }
}

// วิธีที่ 2: ใช้ ViewModel ร่วมกัน (ข้อมูลซับซ้อน)
// SharedViewModel เก็บข้อมูลที่ต้องแชร์ระหว่าง screen
@HiltViewModel
class SharedViewModel @Inject constructor() : ViewModel() {
    var selectedItem by mutableStateOf<Item?>(null)
        private set

    fun selectItem(item: Item) {
        selectedItem = item
    }
}

// ใน NavGraph - ใช้ ViewModel ระดับ NavHost
@Composable
fun AppNavigation() {
    val navController = rememberNavController()

    NavHost(navController, startDestination = "home") {
        composable("home") { entry ->
            val sharedVM: SharedViewModel = hiltViewModel(
                navController.getBackStackEntry("home") // ผูกกับ home entry
            )
            HomeScreen(sharedViewModel = sharedVM)
        }
        composable("detail") { entry ->
            val sharedVM: SharedViewModel = hiltViewModel(
                navController.getBackStackEntry("home")
            )
            DetailScreen(sharedViewModel = sharedVM)
        }
    }
}

// วิธีที่ 3: SavedStateHandle (ใน ViewModel รับ args)
@HiltViewModel
class DetailViewModel @Inject constructor(
    savedStateHandle: SavedStateHandle
) : ViewModel() {
    val itemId: Int = savedStateHandle.get<Int>("itemId") ?: 0
}
```

---

## ขั้นตอนที่ 730: Nested Navigation Graph

```kotlin
import androidx.navigation.NavGraphBuilder
import androidx.navigation.compose.navigation

// แบ่ง navigation เป็น nested graph ตาม feature
fun NavGraphBuilder.authNavGraph(navController: NavController) {
    navigation(
        startDestination = "login",
        route = "auth"
    ) {
        composable("login") {
            LoginScreen(
                onLoginSuccess = {
                    navController.navigate("main") {
                        popUpTo("auth") { inclusive = true }
                    }
                },
                onNavigateToRegister = {
                    navController.navigate("register")
                }
            )
        }
        composable("register") {
            RegisterScreen(
                onRegisterSuccess = {
                    navController.navigate("main") {
                        popUpTo("auth") { inclusive = true }
                    }
                },
                onBack = { navController.popBackStack() }
            )
        }
    }
}

fun NavGraphBuilder.mainNavGraph(navController: NavController) {
    navigation(
        startDestination = Screen.Home.route,
        route = "main"
    ) {
        composable(Screen.Home.route) { HomeScreen(navController) }
        composable(Screen.Profile.route) { ProfileScreen(navController) }
    }
}

// Root NavHost ใช้ nested graphs
@Composable
fun RootNavigation() {
    val navController = rememberNavController()
    val isLoggedIn = false // ดูจาก auth state จริง

    NavHost(
        navController = navController,
        startDestination = if (isLoggedIn) "main" else "auth"
    ) {
        authNavGraph(navController)
        mainNavGraph(navController)
    }
}
```

---

*Part 38 จบแล้ว | ก่อนหน้า: [Part 37](../part37/README.md) | ถัดไป: [Part 39](../part39/README.md)*
