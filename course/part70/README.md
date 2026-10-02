# Part 70: World-Class App Architecture
## ขั้นตอนที่ 1226-1250

---

## ขั้นตอนที่ 1226: Now in Android Architecture

```
"Now in Android" - Google's reference architecture:

:app
├── :feature:foryou
├── :feature:bookmarks
├── :feature:interests
├── :feature:search
├── :feature:settings
│
:core
├── :core:common
├── :core:data        (Repository, DataSource)
├── :core:database    (Room)
├── :core:datastore   (DataStore)
├── :core:domain      (UseCases, Models)
├── :core:model       (Data classes)
├── :core:network     (Retrofit, API)
├── :core:notifications
├── :core:testing     (test utilities)
├── :core:ui          (Compose components)
└── :core:designsystem (Design tokens, components)

สิ่งที่ทำให้ world-class:
1. Strict layering (no layer violations)
2. 100% testable (all dependencies injectable)
3. Offline-first
4. Reactive (UI driven by State flows)
5. Performant (Baseline Profiles, lazy loading)
6. Accessible (semantics, a11y)
```

---

## ขั้นตอนที่ 1227: Design System

```kotlin
// ============================================
// Design System - App-wide design tokens
// ============================================

// :core:designsystem module

// Color tokens
object AppColors {
    // Brand colors
    val Primary = Color(0xFF6C5CE7)
    val PrimaryContainer = Color(0xFFEDE9FF)
    val OnPrimary = Color.White
    val OnPrimaryContainer = Color(0xFF210066)
    
    // Neutral
    val Background = Color(0xFFFFFBFE)
    val Surface = Color(0xFFFFFBFE)
    val SurfaceVariant = Color(0xFFE7E0EC)
    
    // Semantic
    val Success = Color(0xFF00B894)
    val Warning = Color(0xFFFDCB6E)
    val Error = Color(0xFFD63031)
    val Info = Color(0xFF74B9FF)
}

// Typography tokens
object AppTypography {
    val DisplayLarge = TextStyle(
        fontFamily = FontFamily.Default,
        fontWeight = FontWeight.Normal,
        fontSize = 57.sp,
        lineHeight = 64.sp,
        letterSpacing = (-0.25).sp
    )
    
    val TitleLarge = TextStyle(
        fontWeight = FontWeight.SemiBold,
        fontSize = 22.sp,
        lineHeight = 28.sp
    )
    
    val BodyLarge = TextStyle(
        fontWeight = FontWeight.Normal,
        fontSize = 16.sp,
        lineHeight = 24.sp,
        letterSpacing = 0.5.sp
    )
    
    val LabelMedium = TextStyle(
        fontWeight = FontWeight.Medium,
        fontSize = 12.sp,
        lineHeight = 16.sp,
        letterSpacing = 0.5.sp
    )
}

// Spacing tokens
object Spacing {
    val xs = 4.dp
    val sm = 8.dp
    val md = 16.dp
    val lg = 24.dp
    val xl = 32.dp
    val xxl = 48.dp
}

// Elevation tokens
object Elevation {
    val Level0 = 0.dp
    val Level1 = 1.dp
    val Level2 = 3.dp
    val Level3 = 6.dp
    val Level4 = 8.dp
    val Level5 = 12.dp
}

// ============================================
// Design System Components
// ============================================

@Composable
fun AppPrimaryButton(
    text: String,
    onClick: () -> Unit,
    modifier: Modifier = Modifier,
    enabled: Boolean = true,
    loading: Boolean = false,
    leadingIcon: ImageVector? = null
) {
    Button(
        onClick = onClick,
        modifier = modifier.fillMaxWidth().height(48.dp),
        enabled = enabled && !loading,
        colors = ButtonDefaults.buttonColors(
            containerColor = AppColors.Primary,
            disabledContainerColor = AppColors.Primary.copy(alpha = 0.38f)
        ),
        shape = RoundedCornerShape(12.dp)
    ) {
        if (loading) {
            CircularProgressIndicator(
                modifier = Modifier.size(20.dp),
                color = AppColors.OnPrimary,
                strokeWidth = 2.dp
            )
        } else {
            leadingIcon?.let {
                Icon(it, null, modifier = Modifier.size(18.dp))
                Spacer(Modifier.width(Spacing.sm))
            }
            Text(
                text = text,
                style = AppTypography.LabelMedium.copy(fontWeight = FontWeight.SemiBold),
                color = AppColors.OnPrimary
            )
        }
    }
}

@Composable
fun AppCard(
    modifier: Modifier = Modifier,
    onClick: (() -> Unit)? = null,
    elevation: Dp = Elevation.Level1,
    content: @Composable ColumnScope.() -> Unit
) {
    Card(
        modifier = modifier.then(
            if (onClick != null) Modifier.clickable(onClick = onClick) else Modifier
        ),
        elevation = CardDefaults.cardElevation(defaultElevation = elevation),
        shape = RoundedCornerShape(16.dp)
    ) {
        Column(
            modifier = Modifier.padding(Spacing.md),
            content = content
        )
    }
}
```

---

## ขั้นตอนที่ 1228: Advanced State Management

```kotlin
// ============================================
// UI State Machine
// ============================================

// State Machine สำหรับ checkout flow
sealed class CheckoutState {
    object Idle : CheckoutState()
    
    data class AddressInput(
        val address: Address? = null,
        val addressError: String? = null
    ) : CheckoutState()
    
    data class PaymentInput(
        val address: Address,
        val paymentMethod: PaymentMethod? = null,
        val paymentError: String? = null
    ) : CheckoutState()
    
    data class ReviewOrder(
        val address: Address,
        val paymentMethod: PaymentMethod,
        val cart: Cart,
        val coupon: Coupon? = null
    ) : CheckoutState()
    
    object Processing : CheckoutState()
    
    data class Success(val orderId: String, val estimatedDelivery: LocalDate) : CheckoutState()
    
    data class Error(val message: String, val canRetry: Boolean) : CheckoutState()
}

sealed class CheckoutIntent {
    data class SetAddress(val address: Address) : CheckoutIntent()
    data class SetPaymentMethod(val method: PaymentMethod) : CheckoutIntent()
    data class ApplyCoupon(val code: String) : CheckoutIntent()
    object PlaceOrder : CheckoutIntent()
    object Retry : CheckoutIntent()
    object Reset : CheckoutIntent()
}

@HiltViewModel
class CheckoutViewModel @Inject constructor(
    private val cartRepository: CartRepository,
    private val orderRepository: OrderRepository,
    private val couponRepository: CouponRepository
) : ViewModel() {
    
    private val _state = MutableStateFlow<CheckoutState>(CheckoutState.Idle)
    val state: StateFlow<CheckoutState> = _state.asStateFlow()
    
    fun onIntent(intent: CheckoutIntent) {
        viewModelScope.launch {
            _state.value = reduce(state.value, intent)
        }
    }
    
    private suspend fun reduce(current: CheckoutState, intent: CheckoutIntent): CheckoutState {
        return when {
            intent is CheckoutIntent.Reset -> CheckoutState.Idle
            
            intent is CheckoutIntent.SetAddress -> {
                when (current) {
                    is CheckoutState.Idle, is CheckoutState.AddressInput ->
                        CheckoutState.PaymentInput(address = intent.address)
                    else -> current
                }
            }
            
            intent is CheckoutIntent.SetPaymentMethod -> {
                when (current) {
                    is CheckoutState.PaymentInput ->
                        CheckoutState.ReviewOrder(
                            address = current.address,
                            paymentMethod = intent.method,
                            cart = cartRepository.getCart()
                        )
                    else -> current
                }
            }
            
            intent is CheckoutIntent.ApplyCoupon && current is CheckoutState.ReviewOrder -> {
                val coupon = runCatching { couponRepository.validate(intent.code) }.getOrNull()
                current.copy(coupon = coupon)
            }
            
            intent is CheckoutIntent.PlaceOrder && current is CheckoutState.ReviewOrder -> {
                try {
                    CheckoutState.Processing.also { _state.value = it }
                    val order = orderRepository.placeOrder(
                        address = current.address,
                        paymentMethod = current.paymentMethod,
                        cart = current.cart,
                        coupon = current.coupon
                    )
                    CheckoutState.Success(order.id, order.estimatedDelivery)
                } catch (e: Exception) {
                    CheckoutState.Error(e.message ?: "เกิดข้อผิดพลาด", canRetry = true)
                }
            }
            
            intent is CheckoutIntent.Retry && current is CheckoutState.Error -> {
                CheckoutState.ReviewOrder(
                    // restore previous review state
                    address = /* ... */ Address("", "", ""),
                    paymentMethod = PaymentMethod.CreditCard("", ""),
                    cart = cartRepository.getCart()
                )
            }
            
            else -> current
        }
    }
}
```

---

## ขั้นตอนที่ 1229: Offline-First Architecture

```kotlin
// ============================================
// Offline-First with Sync Queue
// ============================================

// Operations ที่รอ sync
@Entity(tableName = "pending_operations")
data class PendingOperation(
    @PrimaryKey val id: String = UUID.randomUUID().toString(),
    val type: OperationType,
    val entityId: String,
    val payload: String,  // JSON
    val createdAt: Long = System.currentTimeMillis(),
    val retryCount: Int = 0,
    val lastError: String? = null
)

enum class OperationType {
    CREATE_ORDER, UPDATE_PROFILE, ADD_REVIEW, CANCEL_ORDER
}

class SyncQueue @Inject constructor(
    private val pendingDao: PendingOperationDao,
    private val networkMonitor: NetworkMonitor
) {
    
    fun enqueue(operation: PendingOperation) = pendingDao.insert(operation)
    
    suspend fun processQueue() {
        if (!networkMonitor.isOnline) return
        
        val pending = pendingDao.getPending()
        pending.forEach { op ->
            try {
                processOperation(op)
                pendingDao.delete(op)
            } catch (e: Exception) {
                pendingDao.update(op.copy(
                    retryCount = op.retryCount + 1,
                    lastError = e.message
                ))
            }
        }
    }
    
    private suspend fun processOperation(op: PendingOperation) {
        when (op.type) {
            OperationType.CREATE_ORDER -> {
                val order = Gson().fromJson(op.payload, OrderDto::class.java)
                orderApiService.createOrder(order)
            }
            OperationType.UPDATE_PROFILE -> {
                val profile = Gson().fromJson(op.payload, ProfileDto::class.java)
                userApiService.updateProfile(profile)
            }
            // handle other types
            else -> throw IllegalArgumentException("Unknown operation: ${op.type}")
        }
    }
}
```

---

## ขั้นตอนที่ 1230: Production Readiness Checklist

```kotlin
// ============================================
// Production Checklist
// ============================================

object ProductionChecklist {
    
    // Performance
    val performance = listOf(
        "✅ Baseline Profiles generated",
        "✅ Paging 3 for large lists",
        "✅ Image loading with Coil (size/cache)",
        "✅ No main thread blocking operations",
        "✅ Memory leak detection (LeakCanary)",
        "✅ R8/ProGuard minification",
        "✅ Strict mode enabled in debug"
    )
    
    // Security
    val security = listOf(
        "✅ HTTPS everywhere",
        "✅ Certificate pinning",
        "✅ Tokens in EncryptedSharedPreferences",
        "✅ No secrets in source code",
        "✅ ProGuard obfuscation",
        "✅ Root detection (optional)",
        "✅ Screenshot prevention (FLAG_SECURE) for sensitive screens"
    )
    
    // Testing
    val testing = listOf(
        "✅ Unit tests > 80% coverage",
        "✅ Integration tests for Repository",
        "✅ UI tests for critical flows",
        "✅ Screenshot tests for design system",
        "✅ Performance tests (Macrobenchmark)",
        "✅ Manual testing on different devices/OS versions"
    )
    
    // Monitoring
    val monitoring = listOf(
        "✅ Crashlytics integrated",
        "✅ Analytics events tracked",
        "✅ Performance monitoring",
        "✅ Network monitoring",
        "✅ Custom crash reports with context"
    )
    
    // Compliance
    val compliance = listOf(
        "✅ Privacy Policy linked",
        "✅ Data deletion flow",
        "✅ PDPA/GDPR compliance",
        "✅ App permissions justified",
        "✅ Accessibility (a11y) tested"
    )
    
    // Play Store
    val playStore = listOf(
        "✅ App signing configured",
        "✅ Store listing complete (screenshots, description)",
        "✅ Content rating filled",
        "✅ Target SDK = latest",
        "✅ 64-bit support",
        "✅ Android 13 notification permission handled"
    )
}
```

---

## แบบฝึกหัด Part 70

```kotlin
// แบบฝึกหัด: สร้าง Production-Ready App Feature

// สร้าง "Nearby Restaurants" feature ที่:
// 1. Clean Architecture (Domain → Data → Presentation)
// 2. Offline-first (cache ใน Room)
// 3. Location-based (แสดงร้านอาหารใกล้ๆ)
// 4. Real-time filter (ประเภทอาหาร, ราคา, rating)
// 5. Paged results
// 6. Unit tested

// Domain Layer
data class Restaurant(
    val id: Long,
    val name: String,
    val cuisine: Cuisine,
    val rating: Float,
    val priceLevel: PriceLevel,
    val distanceMeters: Int,
    val imageUrl: String,
    val isOpen: Boolean
)

enum class Cuisine { THAI, JAPANESE, ITALIAN, CHINESE, AMERICAN, ALL }
enum class PriceLevel { BUDGET, MODERATE, UPSCALE }

data class RestaurantFilter(
    val cuisine: Cuisine = Cuisine.ALL,
    val maxPrice: PriceLevel? = null,
    val minRating: Float = 0f,
    val maxDistanceMeters: Int = 5000,
    val openNow: Boolean = false
)

interface RestaurantRepository {
    fun observeNearby(
        location: LatLng,
        filter: RestaurantFilter
    ): Flow<PagingData<Restaurant>>
    
    suspend fun getById(id: Long): Restaurant
    suspend fun refresh(location: LatLng)
}

// TODO: implement Data layer, ViewModel, and Composable
```

---

*Part 70 จบแล้ว | ก่อนหน้า: [Part 69](../part69/README.md) | ถัดไป: [Part 71](../part71/README.md)*
