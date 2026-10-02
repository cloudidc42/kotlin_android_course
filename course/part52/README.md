# Part 52: MVVM และ MVI Pattern
## ขั้นตอนที่ 776-800

---

## ขั้นตอนที่ 776: MVVM Pattern

```
MVVM (Model-View-ViewModel):

  ┌─────────┐    observes    ┌─────────────┐
  │  View   │ ◄─────────── │  ViewModel  │
  │(Compose)│               │  (State)    │
  └────┬────┘               └──────┬──────┘
       │ actions                   │ calls
       └────────────────────►──────┘
                                   │
                            ┌──────▼──────┐
                            │    Model    │
                            │(Repository) │
                            └─────────────┘
```

---

## ขั้นตอนที่ 777: MVVM ที่สมบูรณ์

```kotlin
// ============================================
// Model - Repository + Data
// ============================================

data class Product(
    val id: Long,
    val name: String,
    val price: Double,
    val description: String,
    val imageUrl: String,
    val category: String,
    val stock: Int
)

interface ProductRepository {
    suspend fun getProducts(category: String?): List<Product>
    suspend fun getProduct(id: Long): Product?
    fun observeProducts(): Flow<List<Product>>
    suspend fun updateStock(id: Long, quantity: Int): Boolean
}

// ============================================
// ViewModel - Business Logic + State
// ============================================

data class ProductListState(
    val products: List<Product> = emptyList(),
    val filteredProducts: List<Product> = emptyList(),
    val selectedCategory: String? = null,
    val searchQuery: String = "",
    val sortOption: SortOption = SortOption.DEFAULT,
    val isLoading: Boolean = false,
    val error: String? = null,
    val cartItemCount: Int = 0
)

enum class SortOption { DEFAULT, PRICE_ASC, PRICE_DESC, NAME_ASC }

sealed class ProductListAction {
    data class Search(val query: String) : ProductListAction()
    data class FilterCategory(val category: String?) : ProductListAction()
    data class Sort(val option: SortOption) : ProductListAction()
    data class AddToCart(val product: Product) : ProductListAction()
    data class RefreshProducts(val force: Boolean = false) : ProductListAction()
    object ClearError : ProductListAction()
}

@HiltViewModel
class ProductListViewModel @Inject constructor(
    private val repository: ProductRepository,
    private val cartService: CartService
) : ViewModel() {
    
    private val _state = MutableStateFlow(ProductListState())
    val state: StateFlow<ProductListState> = _state.asStateFlow()
    
    init {
        loadProducts()
        observeCart()
    }
    
    fun dispatch(action: ProductListAction) {
        when (action) {
            is ProductListAction.Search -> onSearch(action.query)
            is ProductListAction.FilterCategory -> onFilterCategory(action.category)
            is ProductListAction.Sort -> onSort(action.option)
            is ProductListAction.AddToCart -> onAddToCart(action.product)
            is ProductListAction.RefreshProducts -> loadProducts(action.force)
            ProductListAction.ClearError -> clearError()
        }
    }
    
    private fun loadProducts(force: Boolean = false) {
        viewModelScope.launch {
            _state.update { it.copy(isLoading = true, error = null) }
            try {
                repository.observeProducts().collect { products ->
                    _state.update { state ->
                        state.copy(
                            products = products,
                            filteredProducts = applyFilters(products, state),
                            isLoading = false
                        )
                    }
                }
            } catch (e: Exception) {
                _state.update { it.copy(isLoading = false, error = e.message) }
            }
        }
    }
    
    private fun onSearch(query: String) {
        _state.update { state ->
            val filtered = applyFilters(state.products, state.copy(searchQuery = query))
            state.copy(searchQuery = query, filteredProducts = filtered)
        }
    }
    
    private fun onFilterCategory(category: String?) {
        _state.update { state ->
            val filtered = applyFilters(state.products, state.copy(selectedCategory = category))
            state.copy(selectedCategory = category, filteredProducts = filtered)
        }
    }
    
    private fun onSort(option: SortOption) {
        _state.update { state ->
            val sorted = sortProducts(state.filteredProducts, option)
            state.copy(sortOption = option, filteredProducts = sorted)
        }
    }
    
    private fun onAddToCart(product: Product) {
        viewModelScope.launch {
            cartService.addItem(product)
        }
    }
    
    private fun observeCart() {
        viewModelScope.launch {
            cartService.observeItemCount().collect { count ->
                _state.update { it.copy(cartItemCount = count) }
            }
        }
    }
    
    private fun clearError() { _state.update { it.copy(error = null) } }
    
    private fun applyFilters(products: List<Product>, state: ProductListState): List<Product> {
        return products
            .filter { product ->
                (state.selectedCategory == null || product.category == state.selectedCategory) &&
                (state.searchQuery.isBlank() || product.name.contains(state.searchQuery, ignoreCase = true))
            }
            .let { sortProducts(it, state.sortOption) }
    }
    
    private fun sortProducts(products: List<Product>, option: SortOption): List<Product> {
        return when (option) {
            SortOption.DEFAULT -> products
            SortOption.PRICE_ASC -> products.sortedBy { it.price }
            SortOption.PRICE_DESC -> products.sortedByDescending { it.price }
            SortOption.NAME_ASC -> products.sortedBy { it.name }
        }
    }
}
```

---

## ขั้นตอนที่ 778: MVI Pattern

```
MVI (Model-View-Intent):

  ┌──────────────────────────────────────────┐
  │                  View                    │
  └──────┬──────────────────────┬────────────┘
         │ Intent               │ observes
         ▼                      │
  ┌──────────────┐      ┌───────▼───────────┐
  │  Processor   │      │      State        │
  │(ViewModel)   │──────►  (Immutable +     │
  │              │      │   Single source   │
  └──────────────┘      │   of truth)       │
         │              └───────────────────┘
         │ requests
         ▼
  ┌──────────────┐
  │    Model     │
  │ (Repository) │
  └──────────────┘

Key: Unidirectional data flow, Immutable State
```

---

## ขั้นตอนที่ 779: MVI Implementation

```kotlin
// ============================================
// MVI Contract
// ============================================

interface MviIntent
interface MviState
interface MviEffect

// ============================================
// Login MVI
// ============================================

// Intents (User Actions)
sealed class LoginIntent : MviIntent {
    data class EmailChanged(val email: String) : LoginIntent()
    data class PasswordChanged(val password: String) : LoginIntent()
    object LoginClicked : LoginIntent()
    object GoogleLoginClicked : LoginIntent()
    object ForgotPasswordClicked : LoginIntent()
    object ClearError : LoginIntent()
}

// State (UI State - immutable)
data class LoginState(
    val email: String = "",
    val password: String = "",
    val emailError: String? = null,
    val passwordError: String? = null,
    val isLoading: Boolean = false,
    val isLoginEnabled: Boolean = false
) : MviState {
    val hasErrors: Boolean get() = emailError != null || passwordError != null
}

// Effects (One-time events)
sealed class LoginEffect : MviEffect {
    object NavigateToHome : LoginEffect()
    data class NavigateToForgotPassword(val email: String) : LoginEffect()
    data class ShowError(val message: String) : LoginEffect()
    data class ShowToast(val message: String) : LoginEffect()
}

// ============================================
// ViewModel (Processor/Reducer)
// ============================================

@HiltViewModel
class LoginViewModel @Inject constructor(
    private val authRepository: AuthRepository,
    private val validator: InputValidator
) : ViewModel() {
    
    private val _state = MutableStateFlow(LoginState())
    val state: StateFlow<LoginState> = _state.asStateFlow()
    
    private val _effects = Channel<LoginEffect>()
    val effects = _effects.receiveAsFlow()
    
    fun processIntent(intent: LoginIntent) {
        when (intent) {
            is LoginIntent.EmailChanged -> updateEmail(intent.email)
            is LoginIntent.PasswordChanged -> updatePassword(intent.password)
            LoginIntent.LoginClicked -> performLogin()
            LoginIntent.GoogleLoginClicked -> performGoogleLogin()
            is LoginIntent.ForgotPasswordClicked -> goToForgotPassword()
            LoginIntent.ClearError -> clearError()
        }
    }
    
    private fun updateEmail(email: String) {
        _state.update { state ->
            val error = if (email.isNotEmpty() && !validator.isValidEmail(email))
                "รูปแบบ Email ไม่ถูกต้อง" else null
            
            state.copy(
                email = email,
                emailError = error,
                isLoginEnabled = canLogin(email, state.password, error, state.passwordError)
            )
        }
    }
    
    private fun updatePassword(password: String) {
        _state.update { state ->
            val error = if (password.isNotEmpty() && password.length < 6)
                "รหัสผ่านต้องมีอย่างน้อย 6 ตัวอักษร" else null
            
            state.copy(
                password = password,
                passwordError = error,
                isLoginEnabled = canLogin(state.email, password, state.emailError, error)
            )
        }
    }
    
    private fun performLogin() {
        val currentState = _state.value
        
        // Validate
        val emailError = validator.validateEmail(currentState.email)
        val passwordError = validator.validatePassword(currentState.password)
        
        if (emailError != null || passwordError != null) {
            _state.update { it.copy(emailError = emailError, passwordError = passwordError) }
            return
        }
        
        viewModelScope.launch {
            _state.update { it.copy(isLoading = true) }
            
            when (val result = authRepository.login(currentState.email, currentState.password)) {
                is Result.Success -> {
                    _state.update { it.copy(isLoading = false) }
                    _effects.send(LoginEffect.NavigateToHome)
                }
                is Result.Error -> {
                    _state.update { it.copy(isLoading = false) }
                    _effects.send(LoginEffect.ShowError(
                        result.exception.message ?: "เกิดข้อผิดพลาด"
                    ))
                }
                else -> {}
            }
        }
    }
    
    private fun performGoogleLogin() {
        viewModelScope.launch {
            _state.update { it.copy(isLoading = true) }
            // Google Sign-in logic
        }
    }
    
    private fun goToForgotPassword() {
        viewModelScope.launch {
            _effects.send(LoginEffect.NavigateToForgotPassword(_state.value.email))
        }
    }
    
    private fun clearError() {
        _state.update { it.copy(emailError = null, passwordError = null) }
    }
    
    private fun canLogin(
        email: String, password: String,
        emailError: String?, passwordError: String?
    ): Boolean = email.isNotBlank() && password.isNotBlank() &&
        emailError == null && passwordError == null
}
```

---

## ขั้นตอนที่ 780: MVI View (Compose)

```kotlin
@Composable
fun LoginScreen(
    viewModel: LoginViewModel = hiltViewModel(),
    onNavigateHome: () -> Unit,
    onNavigateForgotPassword: (String) -> Unit
) {
    val state by viewModel.state.collectAsStateWithLifecycle()
    val snackbarHostState = remember { SnackbarHostState() }
    
    // Handle effects
    LaunchedEffect(Unit) {
        viewModel.effects.collect { effect ->
            when (effect) {
                LoginEffect.NavigateToHome -> onNavigateHome()
                is LoginEffect.NavigateToForgotPassword ->
                    onNavigateForgotPassword(effect.email)
                is LoginEffect.ShowError ->
                    snackbarHostState.showSnackbar(effect.message)
                is LoginEffect.ShowToast -> { /* show toast */ }
            }
        }
    }
    
    Scaffold(snackbarHost = { SnackbarHost(snackbarHostState) }) { padding ->
        Column(
            modifier = Modifier
                .fillMaxSize()
                .padding(padding)
                .padding(24.dp),
            horizontalAlignment = Alignment.CenterHorizontally,
            verticalArrangement = Arrangement.Center
        ) {
            Text(
                "เข้าสู่ระบบ",
                style = MaterialTheme.typography.headlineLarge
            )
            
            Spacer(Modifier.height(32.dp))
            
            OutlinedTextField(
                value = state.email,
                onValueChange = { viewModel.processIntent(LoginIntent.EmailChanged(it)) },
                label = { Text("Email") },
                isError = state.emailError != null,
                supportingText = state.emailError?.let { { Text(it, color = MaterialTheme.colorScheme.error) } },
                keyboardOptions = KeyboardOptions(keyboardType = KeyboardType.Email),
                modifier = Modifier.fillMaxWidth()
            )
            
            Spacer(Modifier.height(16.dp))
            
            OutlinedTextField(
                value = state.password,
                onValueChange = { viewModel.processIntent(LoginIntent.PasswordChanged(it)) },
                label = { Text("รหัสผ่าน") },
                isError = state.passwordError != null,
                supportingText = state.passwordError?.let { { Text(it, color = MaterialTheme.colorScheme.error) } },
                visualTransformation = PasswordVisualTransformation(),
                keyboardOptions = KeyboardOptions(keyboardType = KeyboardType.Password),
                modifier = Modifier.fillMaxWidth()
            )
            
            Spacer(Modifier.height(24.dp))
            
            Button(
                onClick = { viewModel.processIntent(LoginIntent.LoginClicked) },
                enabled = state.isLoginEnabled && !state.isLoading,
                modifier = Modifier.fillMaxWidth().height(50.dp)
            ) {
                if (state.isLoading) {
                    CircularProgressIndicator(
                        modifier = Modifier.size(24.dp),
                        color = MaterialTheme.colorScheme.onPrimary,
                        strokeWidth = 2.dp
                    )
                } else {
                    Text("เข้าสู่ระบบ")
                }
            }
            
            TextButton(
                onClick = { viewModel.processIntent(LoginIntent.ForgotPasswordClicked) }
            ) {
                Text("ลืมรหัสผ่าน?")
            }
        }
    }
}
```

---

## ขั้นตอนที่ 781: MVVM vs MVI

```
เปรียบเทียบ MVVM และ MVI:

MVVM:
├── State: หลาย StateFlow/LiveData
├── Update: ViewModel functions ถูก call โดย View
├── Predictability: ปานกลาง (หลาย state อาจ inconsistent)
├── Complexity: ต่ำกว่า, เข้าใจง่ายกว่า
└── เหมาะกับ: apps ที่ state ไม่ซับซ้อน

MVI:
├── State: 1 immutable State object
├── Update: ผ่าน Intent → Reducer → State
├── Predictability: สูง (single source of truth)
├── Complexity: สูงกว่า, learning curve
└── เหมาะกับ: apps ที่ state ซับซ้อน, หลาย state interactions

แนะนำ:
- Project ขนาดเล็กถึงกลาง → MVVM
- Project ใหญ่ state ซับซ้อน → MVI
- ทีมที่รู้ Redux/Elm → MVI
```

---

## แบบฝึกหัด Part 52

```kotlin
// แบบฝึกหัด: Implement Registration Form ด้วย MVI

// Intent
sealed class RegisterIntent {
    data class NameChanged(val name: String) : RegisterIntent()
    data class EmailChanged(val email: String) : RegisterIntent()
    data class PasswordChanged(val password: String) : RegisterIntent()
    data class ConfirmPasswordChanged(val password: String) : RegisterIntent()
    object RegisterClicked : RegisterIntent()
    object AcceptTermsToggled : RegisterIntent()
}

// State
data class RegisterState(
    val name: String = "",
    val email: String = "",
    val password: String = "",
    val confirmPassword: String = "",
    val termsAccepted: Boolean = false,
    val nameError: String? = null,
    val emailError: String? = null,
    val passwordError: String? = null,
    val confirmPasswordError: String? = null,
    val isLoading: Boolean = false,
    val isRegisterEnabled: Boolean = false
)

// Effect
sealed class RegisterEffect {
    object NavigateToHome : RegisterEffect()
    data class ShowError(val message: String) : RegisterEffect()
}

// TODO: Implement RegisterViewModel ที่:
// - Validate ทุก field แบบ real-time
// - Check password match
// - isRegisterEnabled = true เมื่อทุก field valid
// - Handle registration API call
// - Emit appropriate effects

class RegisterViewModel @Inject constructor(
    private val authRepository: AuthRepository
) : ViewModel() {
    // TODO: implement
}
```

---

*Part 52 จบแล้ว | ก่อนหน้า: [Part 51](../part51/README.md) | ถัดไป: [Part 53](../part53/README.md)*
