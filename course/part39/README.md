# Part 39: Compose - ViewModel + StateFlow Integration
## ขั้นตอนที่ 751-775

---

## ขั้นตอนที่ 751: ViewModel พื้นฐานกับ Compose

ViewModel ใน Compose ทำงานเหมือนกับ XML แต่ UI อ่านค่าผ่าน State

```kotlin
// build.gradle.kts
dependencies {
    implementation("androidx.lifecycle:lifecycle-viewmodel-compose:2.7.0")
    implementation("androidx.lifecycle:lifecycle-runtime-compose:2.7.0")
}

// ViewModel
import androidx.lifecycle.ViewModel
import kotlinx.coroutines.flow.MutableStateFlow
import kotlinx.coroutines.flow.StateFlow
import kotlinx.coroutines.flow.asStateFlow
import kotlinx.coroutines.flow.update

data class CounterState(
    val count: Int = 0,
    val history: List<String> = emptyList()
)

class CounterViewModel : ViewModel() {
    // MutableStateFlow เก็บ state ภายใน
    private val _state = MutableStateFlow(CounterState())
    // StateFlow เปิดให้ UI อ่าน (read-only)
    val state: StateFlow<CounterState> = _state.asStateFlow()

    fun increment() {
        _state.update { current ->
            current.copy(
                count = current.count + 1,
                history = current.history + "เพิ่ม → ${current.count + 1}"
            )
        }
    }

    fun decrement() {
        _state.update { current ->
            current.copy(
                count = current.count - 1,
                history = current.history + "ลด → ${current.count - 1}"
            )
        }
    }

    fun reset() {
        _state.update { CounterState(history = it.history + "รีเซ็ต → 0") }
    }
}
```

```kotlin
// Composable ที่ใช้ ViewModel
import androidx.lifecycle.compose.collectAsStateWithLifecycle
import androidx.lifecycle.viewmodel.compose.viewModel

@Composable
fun CounterScreen(viewModel: CounterViewModel = viewModel()) {
    // collectAsStateWithLifecycle - ปลอดภัยกว่า collectAsState
    val state by viewModel.state.collectAsStateWithLifecycle()

    Column(
        modifier = Modifier.padding(16.dp),
        horizontalAlignment = Alignment.CenterHorizontally
    ) {
        Text(
            text = "${state.count}",
            style = MaterialTheme.typography.displayLarge
        )

        Row(horizontalArrangement = Arrangement.spacedBy(8.dp)) {
            Button(onClick = { viewModel.decrement() }) { Text("−") }
            Button(onClick = { viewModel.reset() }) { Text("รีเซ็ต") }
            Button(onClick = { viewModel.increment() }) { Text("+") }
        }

        Divider()

        Text("ประวัติ:", style = MaterialTheme.typography.titleMedium)
        LazyColumn {
            items(state.history.reversed()) { entry ->
                Text(entry, style = MaterialTheme.typography.bodySmall)
            }
        }
    }
}
```

---

## ขั้นตอนที่ 752: StateFlow กับ UI Events

```kotlin
import androidx.lifecycle.ViewModel
import androidx.lifecycle.viewModelScope
import kotlinx.coroutines.channels.Channel
import kotlinx.coroutines.flow.*
import kotlinx.coroutines.launch

// UI State
data class LoginState(
    val email: String = "",
    val password: String = "",
    val isLoading: Boolean = false,
    val emailError: String? = null,
    val passwordError: String? = null
)

// UI Event (one-time events)
sealed class LoginEvent {
    object NavigateToHome : LoginEvent()
    data class ShowError(val message: String) : LoginEvent()
}

class LoginViewModel : ViewModel() {
    private val _state = MutableStateFlow(LoginState())
    val state: StateFlow<LoginState> = _state.asStateFlow()

    // Channel สำหรับ one-time events
    private val _events = Channel<LoginEvent>()
    val events = _events.receiveAsFlow()

    fun onEmailChange(email: String) {
        _state.update { it.copy(email = email, emailError = null) }
    }

    fun onPasswordChange(password: String) {
        _state.update { it.copy(password = password, passwordError = null) }
    }

    fun login() {
        val current = _state.value

        // Validation
        if (current.email.isBlank()) {
            _state.update { it.copy(emailError = "กรุณากรอกอีเมล") }
            return
        }
        if (current.password.length < 6) {
            _state.update { it.copy(passwordError = "รหัสผ่านต้องมีอย่างน้อย 6 ตัวอักษร") }
            return
        }

        viewModelScope.launch {
            _state.update { it.copy(isLoading = true) }
            try {
                // จำลอง API call
                kotlinx.coroutines.delay(1500)
                _events.send(LoginEvent.NavigateToHome)
            } catch (e: Exception) {
                _events.send(LoginEvent.ShowError("เข้าสู่ระบบไม่สำเร็จ: ${e.message}"))
            } finally {
                _state.update { it.copy(isLoading = false) }
            }
        }
    }
}
```

```kotlin
@Composable
fun LoginScreen(
    viewModel: LoginViewModel = viewModel(),
    onLoginSuccess: () -> Unit
) {
    val state by viewModel.state.collectAsStateWithLifecycle()
    val snackbarHostState = remember { SnackbarHostState() }

    // จัดการ one-time events
    LaunchedEffect(Unit) {
        viewModel.events.collect { event ->
            when (event) {
                is LoginEvent.NavigateToHome -> onLoginSuccess()
                is LoginEvent.ShowError -> snackbarHostState.showSnackbar(event.message)
            }
        }
    }

    Scaffold(snackbarHost = { SnackbarHost(snackbarHostState) }) { padding ->
        Column(
            modifier = Modifier.padding(padding).padding(24.dp),
            verticalArrangement = Arrangement.spacedBy(16.dp)
        ) {
            Text("เข้าสู่ระบบ", style = MaterialTheme.typography.headlineLarge)

            OutlinedTextField(
                value = state.email,
                onValueChange = viewModel::onEmailChange,
                label = { Text("อีเมล") },
                isError = state.emailError != null,
                supportingText = state.emailError?.let { { Text(it) } }
            )

            OutlinedTextField(
                value = state.password,
                onValueChange = viewModel::onPasswordChange,
                label = { Text("รหัสผ่าน") },
                visualTransformation = PasswordVisualTransformation(),
                isError = state.passwordError != null,
                supportingText = state.passwordError?.let { { Text(it) } }
            )

            Button(
                onClick = { viewModel.login() },
                enabled = !state.isLoading,
                modifier = Modifier.fillMaxWidth()
            ) {
                if (state.isLoading) {
                    CircularProgressIndicator(
                        modifier = Modifier.size(20.dp),
                        color = MaterialTheme.colorScheme.onPrimary
                    )
                } else {
                    Text("เข้าสู่ระบบ")
                }
            }
        }
    }
}
```

---

## ขั้นตอนที่ 753: ViewModel กับ Repository

```kotlin
// Repository
interface ProductRepository {
    fun getProducts(): Flow<List<Product>>
    suspend fun getProductById(id: Int): Product?
    suspend fun addProduct(product: Product)
    suspend fun deleteProduct(id: Int)
}

// ViewModel ใช้ Repository
sealed class ProductUiState {
    object Loading : ProductUiState()
    data class Success(val products: List<Product>) : ProductUiState()
    data class Error(val message: String) : ProductUiState()
}

class ProductViewModel(
    private val repository: ProductRepository
) : ViewModel() {

    // รวม loading + data + error เป็น sealed class
    val uiState: StateFlow<ProductUiState> = repository
        .getProducts()
        .map { products -> ProductUiState.Success(products) }
        .catch { e -> emit(ProductUiState.Error(e.message ?: "เกิดข้อผิดพลาด")) }
        .stateIn(
            scope = viewModelScope,
            started = SharingStarted.WhileSubscribed(5000),
            initialValue = ProductUiState.Loading
        )

    fun deleteProduct(id: Int) {
        viewModelScope.launch {
            repository.deleteProduct(id)
        }
    }
}

@Composable
fun ProductListScreen(viewModel: ProductViewModel) {
    val uiState by viewModel.uiState.collectAsStateWithLifecycle()

    when (val state = uiState) {
        is ProductUiState.Loading -> {
            Box(Modifier.fillMaxSize(), contentAlignment = Alignment.Center) {
                CircularProgressIndicator()
            }
        }
        is ProductUiState.Error -> {
            Box(Modifier.fillMaxSize(), contentAlignment = Alignment.Center) {
                Text("ข้อผิดพลาด: ${state.message}", color = MaterialTheme.colorScheme.error)
            }
        }
        is ProductUiState.Success -> {
            LazyColumn {
                items(state.products, key = { it.id }) { product ->
                    ProductItem(
                        product = product,
                        onDelete = { viewModel.deleteProduct(product.id) }
                    )
                }
            }
        }
    }
}
```

---

## ขั้นตอนที่ 754: MutableState vs StateFlow ใน ViewModel

```kotlin
import androidx.compose.runtime.getValue
import androidx.compose.runtime.mutableStateOf
import androidx.compose.runtime.setValue

// แบบที่ 1: ใช้ MutableState (Compose-first)
class TodoViewModel : ViewModel() {
    // ใช้ได้เลย ไม่ต้อง collect
    var todos by mutableStateOf(listOf<String>())
        private set

    var inputText by mutableStateOf("")
        private set

    fun onInputChange(text: String) {
        inputText = text
    }

    fun addTodo() {
        if (inputText.isNotBlank()) {
            todos = todos + inputText
            inputText = ""
        }
    }

    fun removeTodo(todo: String) {
        todos = todos - todo
    }
}

@Composable
fun TodoScreen(viewModel: TodoViewModel = viewModel()) {
    // ไม่ต้อง collectAsState - ใช้ได้เลย
    Column(modifier = Modifier.padding(16.dp)) {
        Row {
            OutlinedTextField(
                value = viewModel.inputText,
                onValueChange = viewModel::onInputChange,
                modifier = Modifier.weight(1f),
                label = { Text("เพิ่มรายการ") }
            )
            Spacer(Modifier.width(8.dp))
            Button(onClick = { viewModel.addTodo() }) {
                Icon(Icons.Default.Add, null)
            }
        }

        LazyColumn {
            items(viewModel.todos) { todo ->
                Row(
                    modifier = Modifier.fillMaxWidth().padding(8.dp),
                    horizontalArrangement = Arrangement.SpaceBetween
                ) {
                    Text(todo)
                    IconButton(onClick = { viewModel.removeTodo(todo) }) {
                        Icon(Icons.Default.Delete, null)
                    }
                }
            }
        }
    }
}
```

---

## ขั้นตอนที่ 755: ViewModel Factory และ Hilt

```kotlin
// ไม่ใช้ DI - Manual Factory
class ProductDetailViewModel(
    private val repository: ProductRepository,
    private val productId: Int
) : ViewModel() {
    // ...
}

class ProductDetailViewModelFactory(
    private val repository: ProductRepository,
    private val productId: Int
) : ViewModelProvider.Factory {
    override fun <T : ViewModel> create(modelClass: Class<T>): T {
        return ProductDetailViewModel(repository, productId) as T
    }
}

// ใช้ใน Composable
@Composable
fun ProductDetailScreen(productId: Int) {
    val repository = remember { ProductRepositoryImpl() }
    val viewModel: ProductDetailViewModel = viewModel(
        factory = ProductDetailViewModelFactory(repository, productId)
    )
    // ...
}

// ใช้ Hilt - ง่ายกว่ามาก
@HiltViewModel
class ProductDetailViewModelHilt @Inject constructor(
    private val repository: ProductRepository,
    savedStateHandle: SavedStateHandle
) : ViewModel() {
    private val productId: Int = savedStateHandle.get<Int>("productId")!!
    // ...
}

@Composable
fun ProductDetailScreenHilt() {
    // Hilt จัดการ factory ให้อัตโนมัติ
    val viewModel: ProductDetailViewModelHilt = hiltViewModel()
    // ...
}
```

---

*Part 39 จบแล้ว | ก่อนหน้า: [Part 38](../part38/README.md) | ถัดไป: [Part 40](../part40/README.md)*
