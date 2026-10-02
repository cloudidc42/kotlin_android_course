# Part 97: Enterprise Architecture Patterns
## ขั้นตอนที่ 1901-1925

---

## ขั้นตอนที่ 1901: Feature Flags Architecture

```kotlin
// ============================================
// Feature Flag System
// ============================================

// Feature flag definition
enum class Feature(val key: String, val defaultValue: Boolean) {
    NEW_CHECKOUT("feature_new_checkout", false),
    LOYALTY_PROGRAM("feature_loyalty", false),
    AI_RECOMMENDATIONS("feature_ai_recs", false),
    DARK_MODE_V2("feature_dark_mode_v2", true),
    LIVE_CHAT("feature_live_chat", false)
}

interface FeatureFlagService {
    fun isEnabled(feature: Feature): Boolean
    fun observeFlag(feature: Feature): Flow<Boolean>
    suspend fun refreshFlags()
}

// Implementation - delegates to Remote Config
class FirebaseFeatureFlagService @Inject constructor(
    private val remoteConfigManager: RemoteConfigManager
) : FeatureFlagService {
    
    override fun isEnabled(feature: Feature): Boolean {
        return remoteConfigManager.getBoolean(feature.key, feature.defaultValue)
    }
    
    override fun observeFlag(feature: Feature): Flow<Boolean> {
        return remoteConfigManager.observeFeatureFlag(feature.key)
    }
    
    override suspend fun refreshFlags() {
        remoteConfigManager.fetchAndActivate()
    }
}

// Usage in ViewModel
@HiltViewModel
class CheckoutViewModel @Inject constructor(
    private val featureFlags: FeatureFlagService,
    private val orderRepository: OrderRepository
) : ViewModel() {
    
    val useNewCheckout = featureFlags.isEnabled(Feature.NEW_CHECKOUT)
    
    val isLoyaltyVisible: Flow<Boolean> = featureFlags.observeFlag(Feature.LOYALTY_PROGRAM)
}

// Usage in Composable
@Composable
fun CheckoutScreen(viewModel: CheckoutViewModel = hiltViewModel()) {
    
    val isLoyaltyEnabled by viewModel.isLoyaltyVisible.collectAsStateWithLifecycle(false)
    
    if (viewModel.useNewCheckout) {
        NewCheckoutFlow()
    } else {
        LegacyCheckoutFlow()
    }
    
    if (isLoyaltyEnabled) {
        LoyaltyPointsSection()
    }
}
```

---

## ขั้นตอนที่ 1902: Analytics Architecture

```kotlin
// ============================================
// Analytics Abstraction Layer
// ============================================

// Event model
sealed class AnalyticsEvent {
    abstract val name: String
    abstract val params: Map<String, Any>
    
    data class ScreenView(
        val screenName: String,
        val screenClass: String
    ) : AnalyticsEvent() {
        override val name = "screen_view"
        override val params = mapOf(
            "screen_name" to screenName,
            "screen_class" to screenClass
        )
    }
    
    data class ProductView(
        val productId: Long,
        val productName: String,
        val price: Double,
        val category: String
    ) : AnalyticsEvent() {
        override val name = "view_item"
        override val params = mapOf(
            "items" to listOf(mapOf(
                "item_id" to productId,
                "item_name" to productName,
                "price" to price,
                "item_category" to category
            ))
        )
    }
    
    data class AddToCart(
        val productId: Long,
        val productName: String,
        val price: Double,
        val quantity: Int
    ) : AnalyticsEvent() {
        override val name = "add_to_cart"
        override val params = mapOf(
            "currency" to "THB",
            "value" to price * quantity,
            "items" to listOf(mapOf(
                "item_id" to productId,
                "item_name" to productName,
                "quantity" to quantity
            ))
        )
    }
    
    data class Purchase(
        val transactionId: String,
        val revenue: Double,
        val items: List<Map<String, Any>>
    ) : AnalyticsEvent() {
        override val name = "purchase"
        override val params = mapOf(
            "transaction_id" to transactionId,
            "currency" to "THB",
            "value" to revenue,
            "items" to items
        )
    }
    
    data class Custom(
        override val name: String,
        override val params: Map<String, Any> = emptyMap()
    ) : AnalyticsEvent()
}

// Analytics tracker interface
interface AnalyticsTracker {
    fun track(event: AnalyticsEvent)
    fun setUserId(userId: String?)
    fun setUserProperty(name: String, value: String)
}

// Firebase Analytics implementation
class FirebaseAnalyticsTracker @Inject constructor(
    private val firebaseAnalytics: FirebaseAnalytics
) : AnalyticsTracker {
    
    override fun track(event: AnalyticsEvent) {
        val bundle = Bundle().apply {
            event.params.forEach { (key, value) ->
                when (value) {
                    is String -> putString(key, value)
                    is Int -> putInt(key, value)
                    is Long -> putLong(key, value)
                    is Double -> putDouble(key, value)
                    is Boolean -> putBoolean(key, value)
                    is List<*> -> {
                        val items = value.filterIsInstance<Map<*, *>>()
                        putParcelableArrayList(key, ArrayList(items.map { map ->
                            Bundle().apply {
                                map.forEach { (k, v) ->
                                    when (v) {
                                        is String -> putString(k.toString(), v)
                                        is Number -> putDouble(k.toString(), v.toDouble())
                                    }
                                }
                            }
                        }))
                    }
                }
            }
        }
        
        firebaseAnalytics.logEvent(event.name, bundle)
    }
    
    override fun setUserId(userId: String?) {
        firebaseAnalytics.setUserId(userId)
    }
    
    override fun setUserProperty(name: String, value: String) {
        firebaseAnalytics.setUserProperty(name, value)
    }
}

// Composite tracker - send to multiple analytics providers
class CompositeAnalyticsTracker @Inject constructor(
    private val trackers: Set<@JvmSuppressWildcards AnalyticsTracker>
) : AnalyticsTracker {
    
    override fun track(event: AnalyticsEvent) {
        trackers.forEach { it.track(event) }
    }
    
    override fun setUserId(userId: String?) {
        trackers.forEach { it.setUserId(userId) }
    }
    
    override fun setUserProperty(name: String, value: String) {
        trackers.forEach { it.setUserProperty(name, value) }
    }
}
```

---

## ขั้นตอนที่ 1903: State Machine

```kotlin
// ============================================
// Finite State Machine
// ============================================

interface StateMachine<S : Any, E : Any, A : Any> {
    val currentState: StateFlow<S>
    fun dispatch(event: E)
}

// Order state machine
sealed class OrderState {
    object Idle : OrderState()
    data class AddingToCart(val productId: Long) : OrderState()
    data class InCart(val cartId: Long, val items: List<CartItem>) : OrderState()
    object Processing : OrderState()
    data class PaymentPending(val orderId: String, val total: Double) : OrderState()
    data class Success(val orderId: String) : OrderState()
    data class Failed(val error: String) : OrderState()
}

sealed class OrderEvent {
    data class AddProduct(val productId: Long) : OrderEvent()
    object ProceedToCheckout : OrderEvent()
    data class PaymentConfirmed(val paymentToken: String) : OrderEvent()
    object Retry : OrderEvent()
    object Cancel : OrderEvent()
}

class OrderStateMachine @Inject constructor(
    private val orderRepository: OrderRepository,
    private val paymentRepository: PaymentRepository,
    @IoDispatcher private val dispatcher: CoroutineDispatcher
) : StateMachine<OrderState, OrderEvent, Unit> {
    
    private val _state = MutableStateFlow<OrderState>(OrderState.Idle)
    override val currentState: StateFlow<OrderState> = _state.asStateFlow()
    
    private val scope = CoroutineScope(dispatcher + SupervisorJob())
    
    override fun dispatch(event: OrderEvent) {
        scope.launch {
            val currentState = _state.value
            val nextState = transition(currentState, event)
            _state.value = nextState
        }
    }
    
    private suspend fun transition(state: OrderState, event: OrderEvent): OrderState {
        return when {
            state is OrderState.Idle && event is OrderEvent.AddProduct -> {
                _state.value = OrderState.AddingToCart(event.productId)
                try {
                    val cart = orderRepository.addToCart(event.productId)
                    OrderState.InCart(cart.id, cart.items)
                } catch (e: Exception) {
                    OrderState.Failed(e.message ?: "Failed to add to cart")
                }
            }
            
            state is OrderState.InCart && event is OrderEvent.ProceedToCheckout -> {
                _state.value = OrderState.Processing
                try {
                    val order = orderRepository.createOrder(state.cartId)
                    OrderState.PaymentPending(order.id, order.total)
                } catch (e: Exception) {
                    OrderState.Failed(e.message ?: "Failed to create order")
                }
            }
            
            state is OrderState.PaymentPending && event is OrderEvent.PaymentConfirmed -> {
                try {
                    paymentRepository.confirmPayment(state.orderId, event.paymentToken)
                    OrderState.Success(state.orderId)
                } catch (e: Exception) {
                    OrderState.Failed(e.message ?: "Payment failed")
                }
            }
            
            event is OrderEvent.Cancel -> OrderState.Idle
            
            event is OrderEvent.Retry && state is OrderState.Failed -> OrderState.Idle
            
            else -> state  // No transition
        }
    }
}
```

---

## ขั้นตอนที่ 1904: Error Handling Strategy

```kotlin
// ============================================
// Centralized Error Handling
// ============================================

// Global error types
sealed class AppError {
    // Network errors
    data class NetworkError(val message: String, val code: Int? = null) : AppError()
    object NoInternet : AppError()
    object ServerUnavailable : AppError()
    
    // Auth errors
    object Unauthorized : AppError()
    object SessionExpired : AppError()
    
    // Business errors
    data class ValidationError(val fields: Map<String, String>) : AppError()
    data class BusinessError(val code: String, val message: String) : AppError()
    
    // System errors
    object UnknownError : AppError()
}

// Error handler
class ErrorHandler @Inject constructor(
    private val analyticsTracker: AnalyticsTracker,
    private val authRepository: AuthRepository
) {
    
    fun handle(error: AppError): ErrorAction {
        return when (error) {
            is AppError.NetworkError -> {
                analyticsTracker.track(AnalyticsEvent.Custom("network_error", 
                    mapOf("code" to (error.code ?: 0), "message" to error.message)))
                ErrorAction.ShowMessage(getNetworkErrorMessage(error.code))
            }
            
            is AppError.NoInternet -> {
                ErrorAction.ShowOfflineMessage
            }
            
            is AppError.Unauthorized,
            is AppError.SessionExpired -> {
                authRepository.signOut()
                ErrorAction.NavigateToLogin
            }
            
            is AppError.ValidationError -> {
                ErrorAction.ShowValidationErrors(error.fields)
            }
            
            is AppError.BusinessError -> {
                ErrorAction.ShowMessage(error.message)
            }
            
            is AppError.ServerUnavailable -> {
                ErrorAction.ShowMessage("เซิร์ฟเวอร์ไม่พร้อมใช้งาน กรุณาลองใหม่ภายหลัง")
            }
            
            is AppError.UnknownError -> {
                analyticsTracker.track(AnalyticsEvent.Custom("unknown_error"))
                ErrorAction.ShowMessage("เกิดข้อผิดพลาดที่ไม่คาดคิด")
            }
        }
    }
    
    private fun getNetworkErrorMessage(code: Int?): String = when (code) {
        400 -> "ข้อมูลไม่ถูกต้อง"
        403 -> "ไม่มีสิทธิ์เข้าถึง"
        404 -> "ไม่พบข้อมูลที่ต้องการ"
        429 -> "ร้องขอมากเกินไป กรุณารอสักครู่"
        500, 502, 503 -> "เกิดข้อผิดพลาดจากเซิร์ฟเวอร์"
        else -> "เกิดข้อผิดพลาดในการเชื่อมต่อ"
    }
}

sealed class ErrorAction {
    data class ShowMessage(val message: String) : ErrorAction()
    data class ShowValidationErrors(val fields: Map<String, String>) : ErrorAction()
    object ShowOfflineMessage : ErrorAction()
    object NavigateToLogin : ErrorAction()
    data class Retry(val action: suspend () -> Unit) : ErrorAction()
}
```

---

*Part 97 จบแล้ว | ก่อนหน้า: [Part 96](../part96/README.md) | ถัดไป: [Part 98](../part98/README.md)*
