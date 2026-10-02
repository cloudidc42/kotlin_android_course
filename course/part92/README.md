# Part 92: Navigation Compose Advanced
## ขั้นตอนที่ 1776-1800

---

## ขั้นตอนที่ 1776: Type-Safe Navigation

```kotlin
// ============================================
// Type-Safe Routes ด้วย Serializable (API 2.8+)
// ============================================

// Define routes as data classes/objects
@Serializable
object HomeRoute

@Serializable
data class ProductDetailRoute(val productId: Long)

@Serializable
data class ProfileRoute(val userId: Long, val tab: String = "overview")

@Serializable
object CartRoute

@Serializable
data class CheckoutRoute(
    val cartId: Long,
    val couponCode: String? = null
)

// Setup NavHost
@Composable
fun AppNavHost(navController: NavHostController) {
    NavHost(
        navController = navController,
        startDestination = HomeRoute
    ) {
        composable<HomeRoute> {
            HomeScreen(
                onProductClick = { id ->
                    navController.navigate(ProductDetailRoute(id))
                },
                onCartClick = { navController.navigate(CartRoute) }
            )
        }
        
        composable<ProductDetailRoute> { backStackEntry ->
            val route: ProductDetailRoute = backStackEntry.toRoute()
            ProductDetailScreen(
                productId = route.productId,
                onBack = navController::popBackStack,
                onAddToCart = { navController.navigate(CartRoute) }
            )
        }
        
        composable<ProfileRoute> { backStackEntry ->
            val route: ProfileRoute = backStackEntry.toRoute()
            ProfileScreen(
                userId = route.userId,
                initialTab = route.tab,
                onBack = navController::popBackStack
            )
        }
        
        composable<CartRoute> {
            CartScreen(
                onCheckout = { cartId ->
                    navController.navigate(CheckoutRoute(cartId))
                }
            )
        }
        
        composable<CheckoutRoute> { backStackEntry ->
            val route: CheckoutRoute = backStackEntry.toRoute()
            CheckoutScreen(
                cartId = route.cartId,
                couponCode = route.couponCode
            )
        }
    }
}
```

---

## ขั้นตอนที่ 1777: Deep Links

```kotlin
// ============================================
// Deep Links + Implicit Deep Links
// ============================================

// AndroidManifest.xml
/*
<activity android:name=".MainActivity">
    <intent-filter>
        <action android:name="android.intent.action.VIEW" />
        <category android:name="android.intent.category.DEFAULT" />
        <category android:name="android.intent.category.BROWSABLE" />
        <data android:scheme="myapp" android:host="product" />
        <data android:scheme="https" android:host="www.myapp.com" />
    </intent-filter>
</activity>
*/

// Route with deep link support
@Composable
fun AppNavHost(navController: NavHostController) {
    NavHost(
        navController = navController,
        startDestination = HomeRoute
    ) {
        // Legacy string routes for deep links
        composable(
            route = "product/{productId}",
            arguments = listOf(
                navArgument("productId") { type = NavType.LongType }
            ),
            deepLinks = listOf(
                navDeepLink {
                    uriPattern = "myapp://product/{productId}"
                },
                navDeepLink {
                    uriPattern = "https://www.myapp.com/product/{productId}"
                }
            )
        ) { backStackEntry ->
            val productId = backStackEntry.arguments?.getLong("productId") ?: return@composable
            ProductDetailScreen(productId = productId, onBack = navController::popBackStack)
        }
        
        // Type-safe route with deep link
        composable<ProfileRoute>(
            deepLinks = listOf(
                navDeepLink<ProfileRoute>(
                    basePath = "https://www.myapp.com/profile"
                )
            )
        ) { backStackEntry ->
            val route: ProfileRoute = backStackEntry.toRoute()
            ProfileScreen(userId = route.userId, initialTab = route.tab)
        }
    }
}

// Handle incoming deep links in MainActivity
class MainActivity : ComponentActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        
        setContent {
            val navController = rememberNavController()
            
            AppNavHost(navController)
        }
    }
}

// Programmatic deep link navigation
fun NavController.navigateToProductViaDeepLink(productId: Long) {
    val deepLinkUri = Uri.parse("myapp://product/$productId")
    navigate(deepLinkUri)
}
```

---

## ขั้นตอนที่ 1778: Nested Navigation

```kotlin
// ============================================
// Nested NavGraph
// ============================================

// Auth NavGraph
fun NavGraphBuilder.authGraph(navController: NavHostController) {
    navigation(
        route = "auth",
        startDestination = "login"
    ) {
        composable("login") {
            LoginScreen(
                onLoginSuccess = {
                    navController.navigate("main") {
                        popUpTo("auth") { inclusive = true }
                    }
                },
                onRegisterClick = { navController.navigate("register") },
                onForgotPassword = { navController.navigate("forgot_password") }
            )
        }
        
        composable("register") {
            RegisterScreen(
                onRegisterSuccess = {
                    navController.navigate("main") {
                        popUpTo("auth") { inclusive = true }
                    }
                },
                onBack = navController::popBackStack
            )
        }
        
        composable("forgot_password") {
            ForgotPasswordScreen(onBack = navController::popBackStack)
        }
    }
}

// Main NavGraph with Bottom Navigation
fun NavGraphBuilder.mainGraph(navController: NavHostController) {
    navigation(
        route = "main",
        startDestination = "home"
    ) {
        composable("home") { HomeScreen(navController) }
        composable("search") { SearchScreen(navController) }
        composable("cart") { CartScreen(navController) }
        composable("profile") { ProfileScreen(navController) }
    }
}

// Root
@Composable
fun RootNavHost() {
    val navController = rememberNavController()
    
    NavHost(navController = navController, startDestination = "auth") {
        authGraph(navController)
        mainGraph(navController)
    }
}

// ============================================
// Bottom Navigation with NavController
// ============================================

sealed class BottomNavItem(val route: String, val icon: ImageVector, val label: String) {
    object Home : BottomNavItem("home", Icons.Default.Home, "หน้าแรก")
    object Search : BottomNavItem("search", Icons.Default.Search, "ค้นหา")
    object Cart : BottomNavItem("cart", Icons.Default.ShoppingCart, "ตะกร้า")
    object Profile : BottomNavItem("profile", Icons.Default.Person, "โปรไฟล์")
}

@Composable
fun MainScreen() {
    val navController = rememberNavController()
    val navBackStackEntry by navController.currentBackStackEntryAsState()
    val currentRoute = navBackStackEntry?.destination?.route
    
    val items = listOf(
        BottomNavItem.Home,
        BottomNavItem.Search,
        BottomNavItem.Cart,
        BottomNavItem.Profile
    )
    
    val showBottomNav = currentRoute in items.map { it.route }
    
    Scaffold(
        bottomBar = {
            if (showBottomNav) {
                NavigationBar {
                    items.forEach { item ->
                        NavigationBarItem(
                            icon = { Icon(item.icon, contentDescription = item.label) },
                            label = { Text(item.label) },
                            selected = currentRoute == item.route,
                            onClick = {
                                navController.navigate(item.route) {
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
    ) { padding ->
        NavHost(
            navController = navController,
            startDestination = "home",
            modifier = Modifier.padding(padding)
        ) {
            composable("home") { HomeScreen(navController) }
            composable("search") { SearchScreen(navController) }
            composable("cart") { CartScreen(navController) }
            composable("profile") { ProfileScreen(navController) }
        }
    }
}
```

---

## ขั้นตอนที่ 1779: Navigation State Management

```kotlin
// ============================================
// Saving and Restoring Navigation State
// ============================================

// Save state across process death
class NavigationStateHolder @Inject constructor() {
    
    private var savedStateMap = mutableMapOf<String, Bundle>()
    
    fun saveState(navController: NavController) {
        navController.currentBackStack.value.forEach { entry ->
            val bundle = Bundle()
            entry.arguments?.let { bundle.putAll(it) }
            savedStateMap[entry.id] = bundle
        }
    }
    
    fun getState(entryId: String): Bundle? = savedStateMap[entryId]
}

// Back stack result passing
@Composable
fun ProductListScreen(navController: NavController) {
    
    // Listen for result from ProductDetail
    val savedStateHandle = navController.currentBackStackEntry?.savedStateHandle
    val selectedProductId = savedStateHandle?.get<Long>("selectedProductId")
    
    LaunchedEffect(selectedProductId) {
        selectedProductId?.let { id ->
            // Use the result
            highlightProduct(id)
            savedStateHandle?.remove<Long>("selectedProductId")
        }
    }
}

@Composable
fun ProductDetailScreen(navController: NavController) {
    
    fun onAddToWishlist(productId: Long) {
        // Pass result back to previous screen
        navController.previousBackStackEntry
            ?.savedStateHandle
            ?.set("selectedProductId", productId)
        navController.popBackStack()
    }
}

// ============================================
// Animated Navigation Transitions
// ============================================

@Composable
fun AnimatedNavHost(navController: NavHostController) {
    NavHost(
        navController = navController,
        startDestination = "home",
        enterTransition = {
            slideInHorizontally(
                initialOffsetX = { it },
                animationSpec = tween(300)
            )
        },
        exitTransition = {
            slideOutHorizontally(
                targetOffsetX = { -it / 3 },
                animationSpec = tween(300)
            )
        },
        popEnterTransition = {
            slideInHorizontally(
                initialOffsetX = { -it / 3 },
                animationSpec = tween(300)
            )
        },
        popExitTransition = {
            slideOutHorizontally(
                targetOffsetX = { it },
                animationSpec = tween(300)
            )
        }
    ) {
        composable("home") { HomeScreen(navController) }
        
        // Override transition for specific destination
        composable(
            "modal_screen",
            enterTransition = {
                slideInVertically(initialOffsetY = { it }, animationSpec = tween(400))
            },
            exitTransition = {
                slideOutVertically(targetOffsetY = { it }, animationSpec = tween(400))
            }
        ) {
            ModalScreen(navController)
        }
    }
}
```

---

*Part 92 จบแล้ว | ก่อนหน้า: [Part 91](../part91/README.md) | ถัดไป: [Part 93](../part93/README.md)*
