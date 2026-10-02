# Part 49: Android - ViewModel กับ SavedStateHandle
## ขั้นตอนที่ 1001-1025

---

## ขั้นตอนที่ 1001: SavedStateHandle คืออะไร

SavedStateHandle ช่วยให้ ViewModel รักษา state ไว้ได้เมื่อ process killed (เช่น กด Home นานๆ) หรือรับ Navigation arguments

```kotlin
// ทำไมต้อง SavedStateHandle?
// 1. User เปิด app → เข้าหน้า edit form → กด Home
// 2. System kill process (memory pressure)
// 3. User กลับมา → app restart → form ว่างเปล่า!
// SavedStateHandle แก้ปัญหานี้

// ไม่ใช้ SavedStateHandle - state หาย
class BadViewModel : ViewModel() {
    var searchQuery by mutableStateOf("") // หายเมื่อ process killed
}

// ใช้ SavedStateHandle - state อยู่รอด
class GoodViewModel(
    private val savedStateHandle: SavedStateHandle
) : ViewModel() {
    // ใช้ saveable เพื่อ observe เป็น StateFlow
    var searchQuery: StateFlow<String> = savedStateHandle.getStateFlow("search_query", "")

    fun onSearchQueryChange(query: String) {
        savedStateHandle["search_query"] = query
    }
}
```

---

## ขั้นตอนที่ 1002: รับ Navigation Arguments ผ่าน SavedStateHandle

```kotlin
import androidx.lifecycle.SavedStateHandle
import androidx.lifecycle.ViewModel
import androidx.lifecycle.viewModelScope
import kotlinx.coroutines.flow.*
import kotlinx.coroutines.launch

// Route: "product/{productId}?source={source}"
class ProductDetailViewModel(
    private val savedStateHandle: SavedStateHandle,
    private val repository: ProductRepository
) : ViewModel() {

    // รับ argument จาก Navigation
    val productId: Int = savedStateHandle.get<Int>("productId")
        ?: throw IllegalStateException("productId is required")

    val source: String = savedStateHandle.get<String>("source") ?: "unknown"

    // โหลดข้อมูลจาก repository
    val product: StateFlow<ProductDetailUiState> = repository
        .getProductById(productId)
        .map { product ->
            if (product != null) {
                ProductDetailUiState.Success(product)
            } else {
                ProductDetailUiState.Error("ไม่พบสินค้า")
            }
        }
        .catch { e ->
            emit(ProductDetailUiState.Error(e.message ?: "เกิดข้อผิดพลาด"))
        }
        .stateIn(
            scope = viewModelScope,
            started = SharingStarted.WhileSubscribed(5000),
            initialValue = ProductDetailUiState.Loading
        )
}

sealed class ProductDetailUiState {
    object Loading : ProductDetailUiState()
    data class Success(val product: Product) : ProductDetailUiState()
    data class Error(val message: String) : ProductDetailUiState()
}
```

---

## ขั้นตอนที่ 1003: SavedStateHandle กับ Complex State

```kotlin
import androidx.lifecycle.SavedStateHandle
import androidx.lifecycle.ViewModel
import kotlinx.coroutines.flow.StateFlow

// Form ที่รักษา state เมื่อ process killed
class CheckoutViewModel(
    private val savedStateHandle: SavedStateHandle
) : ViewModel() {

    // ทุก field บันทึกลง SavedStateHandle
    val fullName: StateFlow<String> = savedStateHandle.getStateFlow("full_name", "")
    val address: StateFlow<String> = savedStateHandle.getStateFlow("address", "")
    val city: StateFlow<String> = savedStateHandle.getStateFlow("city", "")
    val postalCode: StateFlow<String> = savedStateHandle.getStateFlow("postal_code", "")
    val paymentMethod: StateFlow<String> = savedStateHandle.getStateFlow("payment_method", "credit_card")
    val step: StateFlow<Int> = savedStateHandle.getStateFlow("checkout_step", 1)

    fun updateFullName(name: String) { savedStateHandle["full_name"] = name }
    fun updateAddress(addr: String) { savedStateHandle["address"] = addr }
    fun updateCity(c: String) { savedStateHandle["city"] = c }
    fun updatePostalCode(code: String) { savedStateHandle["postal_code"] = code }
    fun updatePaymentMethod(method: String) { savedStateHandle["payment_method"] = method }

    fun nextStep() {
        val current = step.value
        if (current < 3) savedStateHandle["checkout_step"] = current + 1
    }

    fun previousStep() {
        val current = step.value
        if (current > 1) savedStateHandle["checkout_step"] = current - 1
    }

    fun isCurrentStepValid(): Boolean = when (step.value) {
        1 -> fullName.value.isNotBlank() && address.value.isNotBlank()
        2 -> city.value.isNotBlank() && postalCode.value.length == 5
        3 -> true
        else -> false
    }
}

@Composable
fun CheckoutScreen(viewModel: CheckoutViewModel = hiltViewModel()) {
    val fullName by viewModel.fullName.collectAsStateWithLifecycle()
    val address by viewModel.address.collectAsStateWithLifecycle()
    val city by viewModel.city.collectAsStateWithLifecycle()
    val postalCode by viewModel.postalCode.collectAsStateWithLifecycle()
    val step by viewModel.step.collectAsStateWithLifecycle()

    Column(modifier = Modifier.padding(16.dp)) {
        // Progress indicator
        LinearProgressIndicator(
            progress = step / 3f,
            modifier = Modifier.fillMaxWidth()
        )
        Text("ขั้นตอนที่ $step จาก 3", modifier = Modifier.padding(vertical = 8.dp))

        when (step) {
            1 -> ShippingAddressStep(
                fullName = fullName,
                address = address,
                onFullNameChange = viewModel::updateFullName,
                onAddressChange = viewModel::updateAddress
            )
            2 -> CityPostalStep(
                city = city,
                postalCode = postalCode,
                onCityChange = viewModel::updateCity,
                onPostalCodeChange = viewModel::updatePostalCode
            )
            3 -> PaymentStep(viewModel = viewModel)
        }

        Row(
            horizontalArrangement = Arrangement.SpaceBetween,
            modifier = Modifier.fillMaxWidth().padding(top = 16.dp)
        ) {
            if (step > 1) {
                OutlinedButton(onClick = { viewModel.previousStep() }) {
                    Text("ย้อนกลับ")
                }
            }
            Button(
                onClick = { viewModel.nextStep() },
                enabled = viewModel.isCurrentStepValid()
            ) {
                Text(if (step < 3) "ถัดไป" else "สั่งซื้อ")
            }
        }
    }
}
```

---

## ขั้นตอนที่ 1004: SavedStateHandle กับ Hilt

```kotlin
// เมื่อใช้ Hilt, SavedStateHandle inject อัตโนมัติ
@HiltViewModel
class SearchViewModel @Inject constructor(
    private val savedStateHandle: SavedStateHandle,
    private val repository: SearchRepository
) : ViewModel() {

    // State ที่รอดจาก process death
    val searchQuery: StateFlow<String> = savedStateHandle.getStateFlow("query", "")
    val filterCategory: StateFlow<String> = savedStateHandle.getStateFlow("category", "all")
    val sortBy: StateFlow<String> = savedStateHandle.getStateFlow("sort", "relevance")

    // Result ที่คำนวณจาก state
    @OptIn(ExperimentalCoroutinesApi::class)
    val searchResults: StateFlow<SearchUiState> = combine(
        searchQuery, filterCategory, sortBy
    ) { query, category, sort -> Triple(query, category, sort) }
        .debounce(300)
        .flatMapLatest { (query, category, sort) ->
            if (query.isBlank()) {
                flowOf(SearchUiState.Empty)
            } else {
                repository.search(query, category, sort)
                    .map { results -> SearchUiState.Success(results) }
                    .catch { emit(SearchUiState.Error(it.message ?: "")) }
                    .onStart { emit(SearchUiState.Loading) }
            }
        }
        .stateIn(viewModelScope, SharingStarted.WhileSubscribed(5000), SearchUiState.Empty)

    fun setQuery(query: String) { savedStateHandle["query"] = query }
    fun setCategory(category: String) { savedStateHandle["category"] = category }
    fun setSortBy(sort: String) { savedStateHandle["sort"] = sort }
}

sealed class SearchUiState {
    object Empty : SearchUiState()
    object Loading : SearchUiState()
    data class Success(val results: List<SearchResult>) : SearchUiState()
    data class Error(val message: String) : SearchUiState()
}

@Composable
fun SearchScreen(viewModel: SearchViewModel = hiltViewModel()) {
    val query by viewModel.searchQuery.collectAsStateWithLifecycle()
    val results by viewModel.searchResults.collectAsStateWithLifecycle()
    val category by viewModel.filterCategory.collectAsStateWithLifecycle()

    Column(modifier = Modifier.padding(16.dp)) {
        // Search bar - state อยู่รอดเมื่อ process killed
        OutlinedTextField(
            value = query,
            onValueChange = viewModel::setQuery,
            label = { Text("ค้นหา") },
            leadingIcon = { Icon(Icons.Default.Search, null) },
            trailingIcon = {
                if (query.isNotEmpty()) {
                    IconButton(onClick = { viewModel.setQuery("") }) {
                        Icon(Icons.Default.Clear, null)
                    }
                }
            },
            modifier = Modifier.fillMaxWidth()
        )

        // Category filter
        LazyRow(horizontalArrangement = Arrangement.spacedBy(8.dp)) {
            items(listOf("all", "books", "electronics", "clothing")) { cat ->
                FilterChip(
                    selected = category == cat,
                    onClick = { viewModel.setCategory(cat) },
                    label = { Text(cat) }
                )
            }
        }

        // Results
        when (val state = results) {
            is SearchUiState.Empty -> Text("กรอกคำค้นหาเพื่อเริ่มค้น")
            is SearchUiState.Loading -> CircularProgressIndicator()
            is SearchUiState.Error -> Text(state.message, color = MaterialTheme.colorScheme.error)
            is SearchUiState.Success -> {
                LazyColumn {
                    items(state.results) { result ->
                        ListItem(headlineContent = { Text(result.title) })
                        Divider()
                    }
                }
            }
        }
    }
}
```

---

## ขั้นตอนที่ 1005: Testing ViewModel กับ SavedStateHandle

```kotlin
import androidx.lifecycle.SavedStateHandle
import kotlinx.coroutines.test.*
import org.junit.Test

class SearchViewModelTest {
    private val testDispatcher = UnconfinedTestDispatcher()

    @Test
    fun `searchQuery survives process death simulation`() = runTest {
        // สร้าง SavedStateHandle พร้อม initial state
        val savedStateHandle = SavedStateHandle(mapOf("query" to "kotlin"))

        val viewModel = SearchViewModel(
            savedStateHandle = savedStateHandle,
            repository = FakeSearchRepository()
        )

        // ตรวจสอบว่า state ถูก restore
        assert(viewModel.searchQuery.value == "kotlin")
    }

    @Test
    fun `setQuery updates savedStateHandle`() = runTest {
        val savedStateHandle = SavedStateHandle()
        val viewModel = SearchViewModel(
            savedStateHandle = savedStateHandle,
            repository = FakeSearchRepository()
        )

        viewModel.setQuery("android")

        // ตรวจสอบใน savedStateHandle โดยตรง
        assert(savedStateHandle.get<String>("query") == "android")
    }

    @Test
    fun `categoryFilter persists across recreation`() = runTest {
        val savedStateHandle = SavedStateHandle()
        val viewModel = SearchViewModel(savedStateHandle, FakeSearchRepository())

        viewModel.setCategory("books")

        // จำลอง ViewModel recreation ด้วย state เดิม
        val recreatedViewModel = SearchViewModel(savedStateHandle, FakeSearchRepository())
        assert(recreatedViewModel.filterCategory.value == "books")
    }
}

class FakeSearchRepository : SearchRepository {
    override fun search(query: String, category: String, sort: String): Flow<List<SearchResult>> =
        flowOf(listOf(SearchResult(title = "Kotlin Cookbook"), SearchResult(title = "Android Dev")))
}
```

---

*Part 49 จบแล้ว | ก่อนหน้า: [Part 48](../part48/README.md) | ถัดไป: [Part 50](../part50/README.md)*
