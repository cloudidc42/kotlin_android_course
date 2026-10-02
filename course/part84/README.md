# Part 84: Clean Architecture Deep Dive
## ขั้นตอนที่ 1576-1600

---

## ขั้นตอนที่ 1576: Clean Architecture Principles

```
Clean Architecture (Robert C. Martin):

Concentric Circles:
1. Entities (innermost) - Enterprise Business Rules
   - Data models ที่ไม่ขึ้นกับ framework ใดๆ
   - Plain Kotlin data classes

2. Use Cases - Application Business Rules
   - Interactors
   - Business logic ที่เกี่ยวกับ app นี้

3. Interface Adapters - Controllers, Gateways, Presenters
   - ViewModels
   - Repository interfaces
   - Mappers

4. Frameworks & Drivers (outermost)
   - Android Framework, Room, Retrofit
   - UI (Compose)
   - External libraries

The Dependency Rule:
- Dependencies ชี้เข้าด้านใน เสมอ
- Inner circles ไม่รู้จัก outer circles
- Use Cases ไม่รู้จัก ViewModel
- Entities ไม่รู้จัก Room
```

---

## ขั้นตอนที่ 1577: Domain Layer

```kotlin
// ============================================
// Domain Layer (innermost, no dependencies)
// ============================================

// Entities - Pure Kotlin data classes
data class User(
    val id: UserId,
    val name: String,
    val email: Email,
    val role: UserRole,
    val createdAt: Instant
) {
    // Domain logic
    fun canAccessPremium(): Boolean = role == UserRole.PREMIUM || role == UserRole.ADMIN
    fun isAdmin(): Boolean = role == UserRole.ADMIN
}

// Value Objects - type safety
@JvmInline value class UserId(val value: Long)
@JvmInline value class Email(val value: String) {
    init { require(value.contains("@")) { "Invalid email" } }
}

enum class UserRole { FREE, PREMIUM, ADMIN }

// Domain Errors
sealed class DomainError {
    data class NotFound(val resource: String, val id: Any) : DomainError()
    data class Unauthorized(val action: String) : DomainError()
    data class ValidationError(val field: String, val message: String) : DomainError()
    data class NetworkError(val message: String) : DomainError()
    object Unauthenticated : DomainError()
}

// Result type
sealed class DomainResult<out T> {
    data class Success<T>(val data: T) : DomainResult<T>()
    data class Error(val error: DomainError) : DomainResult<Nothing>()
    
    fun getOrNull(): T? = (this as? Success)?.data
    fun errorOrNull(): DomainError? = (this as? Error)?.error
    
    inline fun <R> map(transform: (T) -> R): DomainResult<R> = when (this) {
        is Success -> Success(transform(data))
        is Error -> this
    }
    
    inline fun onSuccess(action: (T) -> Unit): DomainResult<T> {
        if (this is Success) action(data)
        return this
    }
    
    inline fun onError(action: (DomainError) -> Unit): DomainResult<T> {
        if (this is Error) action(error)
        return this
    }
}
```

---

## ขั้นตอนที่ 1578: Use Cases

```kotlin
// ============================================
// Use Cases - Application Business Logic
// ============================================

// Base use case interface
interface UseCase<in Params, out Result> {
    suspend operator fun invoke(params: Params): Result
}

// Use case สำหรับ get user
class GetUserUseCase @Inject constructor(
    private val userRepository: UserRepository  // interface only
) : UseCase<UserId, DomainResult<User>> {
    
    override suspend operator fun invoke(params: UserId): DomainResult<User> {
        return userRepository.getUserById(params)
    }
}

// Use case สำหรับ update profile พร้อม validation
class UpdateUserProfileUseCase @Inject constructor(
    private val userRepository: UserRepository,
    private val validator: UserValidator
) {
    data class Params(
        val userId: UserId,
        val name: String,
        val bio: String,
        val avatarUri: String?
    )
    
    suspend operator fun invoke(params: Params): DomainResult<User> {
        // Validate
        val nameError = validator.validateName(params.name)
        if (nameError != null) {
            return DomainResult.Error(DomainError.ValidationError("name", nameError))
        }
        
        val bioError = validator.validateBio(params.bio)
        if (bioError != null) {
            return DomainResult.Error(DomainError.ValidationError("bio", bioError))
        }
        
        // Upload avatar if provided
        val avatarUrl = params.avatarUri?.let { uri ->
            when (val result = userRepository.uploadAvatar(params.userId, uri)) {
                is DomainResult.Success -> result.data
                is DomainResult.Error -> return result
            }
        }
        
        // Update profile
        return userRepository.updateProfile(
            userId = params.userId,
            name = params.name,
            bio = params.bio,
            avatarUrl = avatarUrl
        )
    }
}

// Composed Use Case
class GetUserDashboardUseCase @Inject constructor(
    private val getUserUseCase: GetUserUseCase,
    private val getRecentOrdersUseCase: GetRecentOrdersUseCase,
    private val getRecommendationsUseCase: GetRecommendationsUseCase
) {
    data class Dashboard(
        val user: User,
        val recentOrders: List<Order>,
        val recommendations: List<Product>
    )
    
    suspend operator fun invoke(userId: UserId): DomainResult<Dashboard> {
        val userResult = getUserUseCase(userId)
        val user = when (userResult) {
            is DomainResult.Success -> userResult.data
            is DomainResult.Error -> return userResult
        }
        
        // Parallel fetch
        val (ordersDeferred, recommendationsDeferred) = coroutineScope {
            val orders = async { getRecentOrdersUseCase(userId) }
            val recs = async { getRecommendationsUseCase(userId) }
            Pair(orders, recs)
        }
        
        val orders = ordersDeferred.await().getOrNull() ?: emptyList()
        val recommendations = recommendationsDeferred.await().getOrNull() ?: emptyList()
        
        return DomainResult.Success(
            Dashboard(user, orders, recommendations)
        )
    }
}
```

---

## ขั้นตอนที่ 1579: Repository Pattern

```kotlin
// ============================================
// Repository Interface (in domain layer)
// ============================================

interface UserRepository {
    fun observeUser(userId: UserId): Flow<User?>
    suspend fun getUserById(userId: UserId): DomainResult<User>
    suspend fun updateProfile(
        userId: UserId,
        name: String,
        bio: String,
        avatarUrl: String?
    ): DomainResult<User>
    suspend fun uploadAvatar(userId: UserId, localUri: String): DomainResult<String>
}

// ============================================
// Repository Implementation (in data layer)
// ============================================

class UserRepositoryImpl @Inject constructor(
    private val userApi: UserApi,
    private val userDao: UserDao,
    private val networkMonitor: NetworkMonitor,
    private val errorMapper: ApiErrorMapper
) : UserRepository {
    
    override fun observeUser(userId: UserId): Flow<User?> {
        return userDao.observeUser(userId.value)
            .map { entity -> entity?.toDomain() }
    }
    
    override suspend fun getUserById(userId: UserId): DomainResult<User> {
        return try {
            // Offline-first: return cache immediately
            val cached = userDao.getUser(userId.value)?.toDomain()
            
            // Fetch fresh data if online
            if (networkMonitor.isOnline) {
                val response = userApi.getUser(userId.value)
                val user = response.toDomain()
                userDao.upsert(user.toEntity())
                DomainResult.Success(user)
            } else {
                cached?.let { DomainResult.Success(it) }
                    ?: DomainResult.Error(DomainError.NotFound("User", userId))
            }
        } catch (e: HttpException) {
            DomainResult.Error(errorMapper.map(e))
        } catch (e: Exception) {
            DomainResult.Error(DomainError.NetworkError(e.message ?: "Unknown error"))
        }
    }
    
    override suspend fun updateProfile(
        userId: UserId,
        name: String,
        bio: String,
        avatarUrl: String?
    ): DomainResult<User> {
        return try {
            val response = userApi.updateProfile(
                userId.value,
                UpdateProfileRequest(name, bio, avatarUrl)
            )
            val user = response.toDomain()
            userDao.upsert(user.toEntity())
            DomainResult.Success(user)
        } catch (e: Exception) {
            DomainResult.Error(errorMapper.map(e))
        }
    }
    
    override suspend fun uploadAvatar(userId: UserId, localUri: String): DomainResult<String> {
        return try {
            val file = File(localUri)
            val body = RequestBody.create("image/jpeg".toMediaType(), file)
            val part = MultipartBody.Part.createFormData("avatar", file.name, body)
            
            val response = userApi.uploadAvatar(userId.value, part)
            DomainResult.Success(response.avatarUrl)
        } catch (e: Exception) {
            DomainResult.Error(errorMapper.map(e))
        }
    }
}
```

---

## ขั้นตอนที่ 1580: ViewModel Layer

```kotlin
// ============================================
// ViewModel - Interface Adapter Layer
// ============================================

@HiltViewModel
class UserProfileViewModel @Inject constructor(
    savedStateHandle: SavedStateHandle,
    private val getUserUseCase: GetUserUseCase,
    private val updateProfileUseCase: UpdateUserProfileUseCase,
    private val analyticsTracker: AnalyticsTracker
) : ViewModel() {
    
    private val userId = UserId(savedStateHandle.get<Long>("userId") ?: 0L)
    
    private val _uiState = MutableStateFlow(UserProfileUiState())
    val uiState: StateFlow<UserProfileUiState> = _uiState.asStateFlow()
    
    private val _events = Channel<UserProfileEvent>(Channel.BUFFERED)
    val events: Flow<UserProfileEvent> = _events.receiveAsFlow()
    
    init {
        loadUser()
    }
    
    private fun loadUser() {
        viewModelScope.launch {
            _uiState.update { it.copy(isLoading = true, error = null) }
            
            getUserUseCase(userId)
                .onSuccess { user ->
                    _uiState.update { it.copy(isLoading = false, user = user.toUiModel()) }
                    analyticsTracker.track(AnalyticsEvent.ProfileViewed(userId.value))
                }
                .onError { error ->
                    _uiState.update { it.copy(isLoading = false, error = error.toMessage()) }
                }
        }
    }
    
    fun onUpdateProfile(name: String, bio: String) {
        viewModelScope.launch {
            _uiState.update { it.copy(isSaving = true) }
            
            updateProfileUseCase(
                UpdateUserProfileUseCase.Params(userId, name, bio, null)
            )
                .onSuccess { user ->
                    _uiState.update { it.copy(isSaving = false, user = user.toUiModel()) }
                    _events.send(UserProfileEvent.ProfileUpdated)
                }
                .onError { error ->
                    _uiState.update { it.copy(isSaving = false) }
                    _events.send(UserProfileEvent.ShowError(error.toMessage()))
                }
        }
    }
}

data class UserProfileUiState(
    val isLoading: Boolean = false,
    val isSaving: Boolean = false,
    val user: UserUiModel? = null,
    val error: String? = null
)

sealed class UserProfileEvent {
    object ProfileUpdated : UserProfileEvent()
    data class ShowError(val message: String) : UserProfileEvent()
}

// Mapper: Domain → UI model
data class UserUiModel(
    val id: Long,
    val name: String,
    val email: String,
    val avatarUrl: String?,
    val role: String,
    val isPremium: Boolean
)

fun User.toUiModel() = UserUiModel(
    id = id.value,
    name = name,
    email = email.value,
    avatarUrl = null,
    role = role.name,
    isPremium = canAccessPremium()
)

fun DomainError.toMessage(): String = when (this) {
    is DomainError.NotFound -> "ไม่พบข้อมูล: $resource"
    is DomainError.Unauthorized -> "ไม่มีสิทธิ์: $action"
    is DomainError.ValidationError -> "$field: $message"
    is DomainError.NetworkError -> "ข้อผิดพลาดเครือข่าย: $message"
    DomainError.Unauthenticated -> "กรุณาเข้าสู่ระบบ"
}
```

---

## แบบฝึกหัด Part 84

```kotlin
// แบบฝึกหัด: E-Commerce Clean Architecture

// สร้าง feature "Shopping Cart" ด้วย Clean Architecture:
// 
// Domain Layer:
// - Cart entity (CartItem list, discount, total)
// - CartItem entity (Product, quantity, selectedVariant)
// - AddToCartUseCase (validate stock, check limits)
// - RemoveFromCartUseCase
// - ApplyCouponUseCase (validate code, compute discount)
// - GetCartSummaryUseCase (items, subtotal, discount, total, delivery)
//
// Data Layer:
// - CartRepository implementation (Room + API)
// - CartEntity, CartItemEntity
// - Mappers
//
// Presentation Layer:
// - CartViewModel
// - CartUiState
// - CartScreen Compose UI

interface CartRepository {
    fun observeCart(): Flow<Cart>
    suspend fun addItem(productId: Long, variantId: Long?, quantity: Int): DomainResult<Cart>
    suspend fun updateQuantity(itemId: Long, quantity: Int): DomainResult<Cart>
    suspend fun removeItem(itemId: Long): DomainResult<Cart>
    suspend fun applyCoupon(code: String): DomainResult<Cart>
    suspend fun clearCart(): DomainResult<Unit>
}
```

---

*Part 84 จบแล้ว | ก่อนหน้า: [Part 83](../part83/README.md) | ถัดไป: [Part 85](../part85/README.md)*
