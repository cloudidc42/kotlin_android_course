# Part 65: Large-Scale App Architecture
## ขั้นตอนที่ 1101-1125

---

## ขั้นตอนที่ 1101: Scalable Architecture Overview

```
Large-Scale Android App Architecture:

┌─────────────────────────────────────────────────────┐
│                      :app                           │
│  (Entry point, DI setup, top-level navigation)      │
└─────────────┬───────────────────────────────────────┘
              │ depends on
   ┌──────────▼────────────────────────────────┐
   │              Feature Modules               │
   │  :feature:home  :feature:profile           │
   │  :feature:shop  :feature:cart              │
   │  :feature:checkout  :feature:settings      │
   └──────────┬────────────────────────────────┘
              │ depends on
   ┌──────────▼────────────────────────────────┐
   │              Domain Modules                │
   │  :domain:user  :domain:product             │
   │  :domain:order  :domain:cart               │
   │  (Pure Kotlin, no Android deps)            │
   └──────────┬────────────────────────────────┘
              │ depends on
   ┌──────────▼────────────────────────────────┐
   │               Core Modules                 │
   │  :core:network  :core:database             │
   │  :core:ui  :core:common  :core:testing     │
   │  :core:auth  :core:analytics               │
   └────────────────────────────────────────────┘
```

---

## ขั้นตอนที่ 1102: Shared Navigation Architecture

```kotlin
// ============================================
// Type-Safe Navigation ด้วย Sealed Class
// ============================================

// Shared Navigation Destination (ใน :core:navigation)
sealed class Screen(val route: String) {
    // Top-level destinations
    object Home : Screen("home")
    object Profile : Screen("profile")
    object Cart : Screen("cart")
    
    // With arguments
    data class ProductDetail(val productId: Long) : Screen("product/{productId}") {
        val resolvedRoute get() = "product/$productId"
        
        companion object {
            const val ROUTE = "product/{productId}"
            const val ARG_ID = "productId"
        }
    }
    
    data class OrderDetail(val orderId: String) : Screen("order/{orderId}") {
        val resolvedRoute get() = "order/$orderId"
        
        companion object {
            const val ROUTE = "order/{orderId}"
            const val ARG_ID = "orderId"
        }
    }
    
    // Nested graph
    object Auth : Screen("auth")
    object Checkout : Screen("checkout")
}

// ============================================
// Navigation Extensions (ใน :core:navigation)
// ============================================

fun NavController.navigateToProductDetail(productId: Long) {
    navigate(Screen.ProductDetail(productId).resolvedRoute) {
        launchSingleTop = true
    }
}

fun NavController.navigateToOrderDetail(orderId: String) {
    navigate(Screen.OrderDetail(orderId).resolvedRoute) {
        launchSingleTop = true
    }
}

fun NavController.navigateToCart() {
    navigate(Screen.Cart.route) {
        popUpTo(Screen.Home.route)
        launchSingleTop = true
    }
}

fun NavController.navigateToHome() {
    navigate(Screen.Home.route) {
        popUpTo(0) { inclusive = true }
        launchSingleTop = true
    }
}

// ============================================
// AppNavGraph ใน :app
// ============================================

@Composable
fun AppNavGraph(
    navController: NavHostController = rememberNavController(),
    startDestination: String = Screen.Home.route
) {
    NavHost(
        navController = navController,
        startDestination = startDestination
    ) {
        // Feature nav graphs
        homeNavGraph(navController)
        profileNavGraph(navController)
        cartNavGraph(navController)
        checkoutNavGraph(navController)
        
        // Product detail (shared by multiple features)
        composable(
            route = Screen.ProductDetail.ROUTE,
            arguments = listOf(
                navArgument(Screen.ProductDetail.ARG_ID) { type = NavType.LongType }
            ),
            deepLinks = listOf(
                navDeepLink { uriPattern = "https://example.com/products/{productId}" }
            )
        ) { entry ->
            val productId = entry.arguments!!.getLong(Screen.ProductDetail.ARG_ID)
            ProductDetailRoute(
                productId = productId,
                navController = navController
            )
        }
    }
}
```

---

## ขั้นตอนที่ 1103: Feature Flags

```kotlin
// ============================================
// Feature Flags System
// ============================================

interface FeatureFlagProvider {
    fun isEnabled(feature: Feature): Boolean
    suspend fun fetch(): Map<Feature, Boolean>
}

enum class Feature(val key: String, val defaultEnabled: Boolean = false) {
    DARK_MODE("dark_mode", true),
    NEW_CHECKOUT("new_checkout_flow", false),
    AI_RECOMMENDATIONS("ai_recommendations", false),
    SOCIAL_LOGIN("social_login", true),
    LIVE_CHAT("live_chat", false)
}

// Remote Config provider (Firebase)
class FirebaseFeatureFlagProvider @Inject constructor(
    private val remoteConfig: FirebaseRemoteConfig
) : FeatureFlagProvider {
    
    override fun isEnabled(feature: Feature): Boolean {
        return remoteConfig.getBoolean(feature.key)
    }
    
    override suspend fun fetch(): Map<Feature, Boolean> {
        remoteConfig.fetchAndActivate().await()
        return Feature.values().associateWith { isEnabled(it) }
    }
    
    fun getStringValue(key: String): String = remoteConfig.getString(key)
    fun getLongValue(key: String): Long = remoteConfig.getLong(key)
}

// Local fallback
class LocalFeatureFlagProvider @Inject constructor(
    private val prefs: SharedPreferences
) : FeatureFlagProvider {
    
    override fun isEnabled(feature: Feature): Boolean {
        return prefs.getBoolean(feature.key, feature.defaultEnabled)
    }
    
    override suspend fun fetch(): Map<Feature, Boolean> {
        return Feature.values().associateWith { isEnabled(it) }
    }
    
    fun override(feature: Feature, enabled: Boolean) {
        prefs.edit().putBoolean(feature.key, enabled).apply()
    }
    
    fun reset(feature: Feature) {
        prefs.edit().remove(feature.key).apply()
    }
}

// Composite (Remote + Local override for testing)
class CompositeFeatureFlagProvider @Inject constructor(
    private val remote: FirebaseFeatureFlagProvider,
    private val local: LocalFeatureFlagProvider,
    private val buildConfig: BuildConfigProvider
) : FeatureFlagProvider {
    
    override fun isEnabled(feature: Feature): Boolean {
        // Debug builds: allow local override
        if (buildConfig.isDebug && local.hasOverride(feature)) {
            return local.isEnabled(feature)
        }
        return remote.isEnabled(feature)
    }
    
    override suspend fun fetch(): Map<Feature, Boolean> = remote.fetch()
}

// Usage in Composable
@Composable
fun ConditionalFeature(
    feature: Feature,
    featureFlags: FeatureFlagProvider = LocalFeatureFlags.current,
    content: @Composable () -> Unit
) {
    if (featureFlags.isEnabled(feature)) {
        content()
    }
}

val LocalFeatureFlags = staticCompositionLocalOf<FeatureFlagProvider> {
    error("No FeatureFlagProvider provided")
}
```

---

## ขั้นตอนที่ 1104: Analytics Architecture

```kotlin
// ============================================
// Analytics Event System
// ============================================

// Domain Events
sealed class AnalyticsEvent {
    abstract val name: String
    abstract val properties: Map<String, Any>
    
    // User events
    data class UserLogin(val method: String) : AnalyticsEvent() {
        override val name = "user_login"
        override val properties = mapOf("method" to method)
    }
    
    data class ProductViewed(
        val productId: Long,
        val productName: String,
        val price: Double,
        val source: String
    ) : AnalyticsEvent() {
        override val name = "product_viewed"
        override val properties = mapOf(
            "product_id" to productId,
            "product_name" to productName,
            "price" to price,
            "source" to source
        )
    }
    
    data class PurchaseCompleted(
        val orderId: String,
        val totalAmount: Double,
        val itemCount: Int,
        val paymentMethod: String
    ) : AnalyticsEvent() {
        override val name = "purchase"
        override val properties = mapOf(
            "order_id" to orderId,
            "value" to totalAmount,
            "item_count" to itemCount,
            "payment_method" to paymentMethod,
            "currency" to "THB"
        )
    }
    
    data class SearchPerformed(val query: String, val resultCount: Int) : AnalyticsEvent() {
        override val name = "search"
        override val properties = mapOf("query" to query, "result_count" to resultCount)
    }
}

// Analytics tracker interface
interface AnalyticsTracker {
    fun track(event: AnalyticsEvent)
    fun setUserProperty(key: String, value: String)
    fun setUserId(id: String?)
}

// Firebase implementation
class FirebaseAnalyticsTracker @Inject constructor(
    private val analytics: FirebaseAnalytics
) : AnalyticsTracker {
    
    override fun track(event: AnalyticsEvent) {
        val bundle = Bundle().apply {
            event.properties.forEach { (key, value) ->
                when (value) {
                    is String -> putString(key, value)
                    is Long -> putLong(key, value)
                    is Int -> putInt(key, value)
                    is Double -> putDouble(key, value)
                    is Boolean -> putBoolean(key, value)
                }
            }
        }
        analytics.logEvent(event.name, bundle)
    }
    
    override fun setUserProperty(key: String, value: String) {
        analytics.setUserProperty(key, value)
    }
    
    override fun setUserId(id: String?) {
        analytics.setUserId(id)
    }
}
```

---

## แบบฝึกหัด Part 65

```kotlin
// แบบฝึกหัด: สร้าง Event-Driven Architecture สำหรับ Cart

// Events
sealed class CartEvent {
    data class ItemAdded(val productId: Long, val quantity: Int) : CartEvent()
    data class ItemRemoved(val productId: Long) : CartEvent()
    data class QuantityChanged(val productId: Long, val newQuantity: Int) : CartEvent()
    object CartCleared : CartEvent()
    data class CheckoutStarted(val cartTotal: Double) : CartEvent()
}

// State
data class CartState(
    val items: List<CartItem> = emptyList(),
    val total: Double = 0.0,
    val itemCount: Int = 0,
    val isLoading: Boolean = false,
    val error: String? = null
)

// TODO: สร้าง CartViewModel ที่:
// 1. รับ CartEvent
// 2. Update CartState
// 3. Track analytics สำหรับแต่ละ event
// 4. Sync กับ backend (debounced)
// 5. Persist ใน Room

class CartViewModel @Inject constructor(
    private val cartRepository: CartRepository,
    private val analytics: AnalyticsTracker
) : ViewModel() {
    
    private val _state = MutableStateFlow(CartState())
    val state: StateFlow<CartState> = _state.asStateFlow()
    
    fun onEvent(event: CartEvent) {
        viewModelScope.launch {
            // TODO: handle each event
            when (event) {
                is CartEvent.ItemAdded -> TODO()
                is CartEvent.ItemRemoved -> TODO()
                else -> TODO()
            }
        }
    }
}
```

---

*Part 65 จบแล้ว | ก่อนหน้า: [Part 64](../part64/README.md) | ถัดไป: [Part 66](../part66/README.md)*
