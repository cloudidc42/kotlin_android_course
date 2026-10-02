# Part 85: Jetpack Compose Advanced Patterns
## ขั้นตอนที่ 1601-1625

---

## ขั้นตอนที่ 1601: Compound Components Pattern

```kotlin
// ============================================
// Compound Components - flexible API design
// ============================================

// Traditional approach (rigid):
@Composable
fun Card(title: String, subtitle: String, icon: ImageVector, onClick: () -> Unit) {
    // Fixed layout - can't customize
}

// Compound Components approach (flexible):
@Composable
fun Card(
    modifier: Modifier = Modifier,
    onClick: (() -> Unit)? = null,
    content: @Composable CardScope.() -> Unit
) {
    val scope = remember { CardScopeImpl() }
    
    Box(
        modifier = modifier
            .then(if (onClick != null) Modifier.clickable(onClick = onClick) else Modifier)
            .background(MaterialTheme.colorScheme.surface, RoundedCornerShape(16.dp))
            .border(1.dp, MaterialTheme.colorScheme.outlineVariant, RoundedCornerShape(16.dp))
            .padding(16.dp)
    ) {
        scope.content()
    }
}

interface CardScope {
    @Composable fun Header(content: @Composable RowScope.() -> Unit)
    @Composable fun Body(content: @Composable ColumnScope.() -> Unit)
    @Composable fun Footer(content: @Composable RowScope.() -> Unit)
    @Composable fun Divider()
}

private class CardScopeImpl : CardScope {
    @Composable
    override fun Header(content: @Composable RowScope.() -> Unit) {
        Row(
            modifier = Modifier.fillMaxWidth(),
            verticalAlignment = Alignment.CenterVertically,
            content = content
        )
    }
    
    @Composable
    override fun Body(content: @Composable ColumnScope.() -> Unit) {
        Column(
            modifier = Modifier
                .fillMaxWidth()
                .padding(vertical = 8.dp),
            content = content
        )
    }
    
    @Composable
    override fun Footer(content: @Composable RowScope.() -> Unit) {
        Row(
            modifier = Modifier.fillMaxWidth(),
            horizontalArrangement = Arrangement.End,
            content = content
        )
    }
    
    @Composable
    override fun Divider() {
        HorizontalDivider(modifier = Modifier.padding(vertical = 8.dp))
    }
}

// Usage - very flexible
@Composable
fun UserCard(user: User) {
    Card(onClick = { /* navigate */ }) {
        Header {
            AsyncImage(model = user.avatarUrl, contentDescription = null,
                modifier = Modifier.size(48.dp).clip(CircleShape))
            Spacer(Modifier.width(12.dp))
            Column {
                Text(user.name, fontWeight = FontWeight.Bold)
                Text(user.email, style = MaterialTheme.typography.bodySmall)
            }
        }
        Divider()
        Body {
            Text(user.bio)
        }
        Footer {
            TextButton(onClick = {}) { Text("ดูโปรไฟล์") }
            Button(onClick = {}) { Text("ติดตาม") }
        }
    }
}
```

---

## ขั้นตอนที่ 1602: State Hoisting Patterns

```kotlin
// ============================================
// State Hoisting - lifting state for testability
// ============================================

// Stateful (hard to test, less reusable)
@Composable
fun StatefulSearchBar() {
    var query by remember { mutableStateOf("") }
    var results by remember { mutableStateOf<List<String>>(emptyList()) }
    
    // Logic mixed with UI
    SearchBarUI(
        query = query,
        results = results,
        onQueryChange = { newQuery ->
            query = newQuery
            results = search(newQuery)  // search in composable - BAD
        }
    )
}

// Stateless (testable, reusable)
@Composable
fun StatelessSearchBar(
    query: String,
    results: List<String>,
    onQueryChange: (String) -> Unit,
    modifier: Modifier = Modifier
) {
    Column(modifier = modifier) {
        OutlinedTextField(
            value = query,
            onValueChange = onQueryChange,
            placeholder = { Text("ค้นหา...") },
            leadingIcon = { Icon(Icons.Default.Search, null) }
        )
        
        LazyColumn {
            items(results) { result ->
                Text(result, modifier = Modifier.padding(8.dp))
            }
        }
    }
}

// Compose State holder (รวม logic ออกจาก composable)
@Stable
class SearchBarState(
    initialQuery: String = "",
    private val searchFunction: suspend (String) -> List<String>
) {
    var query by mutableStateOf(initialQuery)
        private set
    
    var results by mutableStateOf<List<String>>(emptyList())
        private set
    
    var isLoading by mutableStateOf(false)
        private set
    
    private var searchJob: Job? = null
    
    fun onQueryChange(newQuery: String, scope: CoroutineScope) {
        query = newQuery
        searchJob?.cancel()
        
        if (newQuery.isBlank()) {
            results = emptyList()
            return
        }
        
        searchJob = scope.launch {
            delay(300)  // debounce
            isLoading = true
            results = searchFunction(newQuery)
            isLoading = false
        }
    }
    
    fun clearQuery() {
        query = ""
        results = emptyList()
    }
}

@Composable
fun rememberSearchBarState(
    searchFunction: suspend (String) -> List<String>
): SearchBarState {
    return remember { SearchBarState(searchFunction = searchFunction) }
}

// Usage
@Composable
fun SearchScreen(viewModel: SearchViewModel = hiltViewModel()) {
    val scope = rememberCoroutineScope()
    val state = rememberSearchBarState { query ->
        viewModel.search(query)
    }
    
    StatelessSearchBar(
        query = state.query,
        results = state.results,
        onQueryChange = { state.onQueryChange(it, scope) }
    )
}
```

---

## ขั้นตอนที่ 1603: Modular Compose Architecture

```kotlin
// ============================================
// Feature Architecture: ViewModel → Events → Screen
// ============================================

// MVI Pattern ใน Compose
interface MviViewModel<State, Intent, Event> {
    val state: StateFlow<State>
    val events: Flow<Event>
    fun dispatch(intent: Intent)
}

// Example: Product List Feature
data class ProductListState(
    val products: List<ProductUiModel> = emptyList(),
    val isLoading: Boolean = false,
    val error: String? = null,
    val filter: ProductFilter = ProductFilter(),
    val selectedCount: Int = 0
)

sealed class ProductListIntent {
    object Refresh : ProductListIntent()
    data class Search(val query: String) : ProductListIntent()
    data class FilterByCategory(val category: String) : ProductListIntent()
    data class ToggleSelect(val productId: Long) : ProductListIntent()
    object AddSelectedToCart : ProductListIntent()
}

sealed class ProductListEvent {
    data class NavigateToDetail(val productId: Long) : ProductListEvent()
    data class ShowSnackbar(val message: String) : ProductListEvent()
    object ScrollToTop : ProductListEvent()
}

@HiltViewModel
class ProductListViewModel @Inject constructor(
    private val getProductsUseCase: GetProductsUseCase,
    private val addToCartUseCase: AddToCartUseCase
) : ViewModel(), MviViewModel<ProductListState, ProductListIntent, ProductListEvent> {
    
    private val _state = MutableStateFlow(ProductListState())
    override val state = _state.asStateFlow()
    
    private val _events = Channel<ProductListEvent>(Channel.BUFFERED)
    override val events = _events.receiveAsFlow()
    
    private val selectedIds = mutableSetOf<Long>()
    
    override fun dispatch(intent: ProductListIntent) {
        when (intent) {
            is ProductListIntent.Refresh -> refresh()
            is ProductListIntent.Search -> search(intent.query)
            is ProductListIntent.FilterByCategory -> filterByCategory(intent.category)
            is ProductListIntent.ToggleSelect -> toggleSelect(intent.productId)
            is ProductListIntent.AddSelectedToCart -> addSelectedToCart()
        }
    }
    
    private fun refresh() {
        viewModelScope.launch {
            _state.update { it.copy(isLoading = true, error = null) }
            
            getProductsUseCase(Unit)
                .onSuccess { products ->
                    _state.update { it.copy(
                        products = products.map { p -> p.toUiModel() },
                        isLoading = false
                    ) }
                    _events.send(ProductListEvent.ScrollToTop)
                }
                .onError { error ->
                    _state.update { it.copy(isLoading = false, error = error.toMessage()) }
                }
        }
    }
    
    private fun toggleSelect(id: Long) {
        if (id in selectedIds) selectedIds.remove(id)
        else selectedIds.add(id)
        _state.update { it.copy(selectedCount = selectedIds.size) }
    }
    
    private fun addSelectedToCart() {
        viewModelScope.launch {
            selectedIds.forEach { id ->
                addToCartUseCase(AddToCartUseCase.Params(id, 1))
            }
            selectedIds.clear()
            _state.update { it.copy(selectedCount = 0) }
            _events.send(ProductListEvent.ShowSnackbar("เพิ่มลงตะกร้าแล้ว ${selectedIds.size} รายการ"))
        }
    }
    
    private fun search(query: String) { /* ... */ }
    private fun filterByCategory(category: String) { /* ... */ }
}
```

---

## ขั้นตอนที่ 1604: Custom Layout API

```kotlin
// ============================================
// Custom Layout ด้วย Layout composable
// ============================================

// Staggered Grid Layout
@Composable
fun StaggeredGrid(
    columns: Int = 2,
    modifier: Modifier = Modifier,
    content: @Composable () -> Unit
) {
    Layout(
        content = content,
        modifier = modifier
    ) { measurables, constraints ->
        val columnWidth = constraints.maxWidth / columns
        val columnHeights = IntArray(columns) { 0 }
        
        // Measure children
        val placeables = measurables.mapIndexed { index, measurable ->
            val column = index % columns
            measurable.measure(
                constraints.copy(
                    minWidth = columnWidth,
                    maxWidth = columnWidth
                )
            )
        }
        
        // Calculate positions
        val positions = placeables.mapIndexed { index, placeable ->
            val column = index % columns
            val y = columnHeights[column]
            columnHeights[column] += placeable.height
            Pair(column * columnWidth, y)
        }
        
        val totalHeight = columnHeights.max()
        
        layout(constraints.maxWidth, totalHeight) {
            placeables.forEachIndexed { index, placeable ->
                val (x, y) = positions[index]
                placeable.placeRelative(x, y)
            }
        }
    }
}

// Usage
@Composable
fun PhotoGrid(photos: List<Photo>) {
    StaggeredGrid(columns = 2) {
        photos.forEach { photo ->
            AsyncImage(
                model = photo.url,
                contentDescription = null,
                modifier = Modifier
                    .fillMaxWidth()
                    .height(Random.nextInt(150, 300).dp)
            )
        }
    }
}
```

---

## แบบฝึกหัด Part 85

```kotlin
// แบบฝึกหัด: Rating & Review Component

// สร้าง ReviewComponent ใช้ Compound Component pattern:
// - ReviewComponent { ... }
//   - ReviewSummary(rating, count)
//   - ReviewDistribution(starCounts)
//   - ReviewList(reviews)
//   - WriteReviewButton(onClick)

interface ReviewScope {
    @Composable fun Summary(rating: Float, reviewCount: Int)
    @Composable fun Distribution(starCounts: Map<Int, Int>)
    @Composable fun ReviewList(reviews: List<Review>)
    @Composable fun WriteReviewButton(onClick: () -> Unit)
}

@Composable
fun ReviewComponent(
    modifier: Modifier = Modifier,
    content: @Composable ReviewScope.() -> Unit
) {
    // TODO: implement
}

// Usage
@Composable
fun ProductReviews(product: Product, reviews: List<Review>) {
    ReviewComponent {
        Summary(rating = product.avgRating, reviewCount = reviews.size)
        Distribution(starCounts = reviews.groupBy { it.rating.roundToInt() }.mapValues { it.value.size })
        ReviewList(reviews = reviews.take(5))
        WriteReviewButton(onClick = { /* navigate */ })
    }
}
```

---

*Part 85 จบแล้ว | ก่อนหน้า: [Part 84](../part84/README.md) | ถัดไป: [Part 86](../part86/README.md)*
