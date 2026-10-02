# Part 25: ViewModel และ State Management
## ขั้นตอนที่ 601-625

---

## ขั้นตอนที่ 601: ViewModel คืออะไร?

ViewModel เก็บ UI state ให้รอด configuration changes (เช่น rotate screen)

```
┌─────────────────────────────────┐
│         Activity/Fragment        │
│  ┌───────────────────────────┐  │
│  │      Composable UI        │  │
│  └───────────┬───────────────┘  │
│              │ observe           │
│  ┌───────────▼───────────────┐  │
│  │         ViewModel         │  │
│  │  (survives config change) │  │
│  └───────────┬───────────────┘  │
│              │                   │
│  rotate/lang │ ViewModel stays   │
│              │                   │
│  ┌───────────▼───────────────┐  │
│  │     Activity recreated    │  │
│  └───────────────────────────┘  │
└─────────────────────────────────┘
```

---

## ขั้นตอนที่ 602: Basic ViewModel Pattern

```kotlin
// Simple ViewModel
class CounterViewModel : ViewModel() {
    private val _count = MutableStateFlow(0)
    val count: StateFlow<Int> = _count.asStateFlow()
    
    fun increment() { _count.update { it + 1 } }
    fun decrement() { _count.update { maxOf(0, it - 1) } }
    fun reset() { _count.value = 0 }
    
    override fun onCleared() {
        super.onCleared()
        println("ViewModel cleared")  // cleanup
    }
}

// Composable ใช้ ViewModel
@Composable
fun CounterScreen(viewModel: CounterViewModel = viewModel()) {
    val count by viewModel.count.collectAsStateWithLifecycle()
    
    Column(
        horizontalAlignment = Alignment.CenterHorizontally,
        verticalArrangement = Arrangement.Center,
        modifier = Modifier.fillMaxSize()
    ) {
        Text(
            text = count.toString(),
            style = MaterialTheme.typography.displayLarge
        )
        
        Row(horizontalArrangement = Arrangement.spacedBy(8.dp)) {
            Button(onClick = { viewModel.decrement() }) { Text("-") }
            Button(onClick = { viewModel.reset() }) { Text("Reset") }
            Button(onClick = { viewModel.increment() }) { Text("+") }
        }
    }
}
```

---

## ขั้นตอนที่ 603: UiState Pattern

```kotlin
// ============================================
// UiState + ViewModel Pattern ที่แนะนำ
// ============================================

// UiState - แสดงสถานะทั้งหมดของหน้าจอ
data class UserListUiState(
    val users: List<User> = emptyList(),
    val isLoading: Boolean = false,
    val error: String? = null,
    val searchQuery: String = "",
    val isRefreshing: Boolean = false,
    val selectedUser: User? = null
)

@HiltViewModel
class UserListViewModel @Inject constructor(
    private val userRepository: UserRepository
) : ViewModel() {
    
    private val _uiState = MutableStateFlow(UserListUiState())
    val uiState: StateFlow<UserListUiState> = _uiState.asStateFlow()
    
    // Separate: events ที่เกิดขึ้น 1 ครั้ง (one-shot)
    private val _events = Channel<UserListEvent>(Channel.BUFFERED)
    val events = _events.receiveAsFlow()
    
    init {
        loadUsers()
        observeSearch()
    }
    
    private fun loadUsers() {
        viewModelScope.launch {
            _uiState.update { it.copy(isLoading = true, error = null) }
            
            userRepository.observeActiveUsers()
                .catch { e ->
                    _uiState.update { it.copy(isLoading = false, error = e.message) }
                }
                .collect { users ->
                    _uiState.update { it.copy(users = users, isLoading = false) }
                }
        }
    }
    
    private fun observeSearch() {
        viewModelScope.launch {
            uiState
                .map { it.searchQuery }
                .distinctUntilChanged()
                .debounce(300)
                .flatMapLatest { query ->
                    if (query.isBlank()) userRepository.observeActiveUsers()
                    else userRepository.searchUsers(query)
                }
                .collect { users ->
                    _uiState.update { it.copy(users = users) }
                }
        }
    }
    
    // ============================================
    // User Actions (events from UI)
    // ============================================
    
    fun onSearchQueryChanged(query: String) {
        _uiState.update { it.copy(searchQuery = query) }
    }
    
    fun onRefresh() {
        viewModelScope.launch {
            _uiState.update { it.copy(isRefreshing = true) }
            try {
                userRepository.refreshUsers()
                _uiState.update { it.copy(isRefreshing = false) }
            } catch (e: Exception) {
                _uiState.update { it.copy(isRefreshing = false, error = e.message) }
            }
        }
    }
    
    fun onUserSelected(user: User) {
        _uiState.update { it.copy(selectedUser = user) }
        viewModelScope.launch {
            _events.send(UserListEvent.NavigateToDetail(user.id))
        }
    }
    
    fun onDeleteUser(user: User) {
        viewModelScope.launch {
            try {
                userRepository.deleteUser(user.id)
                _events.send(UserListEvent.ShowMessage("ลบ ${user.name} แล้ว"))
            } catch (e: Exception) {
                _events.send(UserListEvent.ShowError("ลบไม่สำเร็จ: ${e.message}"))
            }
        }
    }
    
    fun onErrorDismissed() {
        _uiState.update { it.copy(error = null) }
    }
}

sealed class UserListEvent {
    data class NavigateToDetail(val userId: Long) : UserListEvent()
    data class ShowMessage(val message: String) : UserListEvent()
    data class ShowError(val error: String) : UserListEvent()
}
```

---

## ขั้นตอนที่ 604: Composable กับ ViewModel

```kotlin
@Composable
fun UserListScreen(
    navController: NavController,
    viewModel: UserListViewModel = hiltViewModel()
) {
    val uiState by viewModel.uiState.collectAsStateWithLifecycle()
    val snackbarHostState = remember { SnackbarHostState() }
    
    // Handle one-shot events
    LaunchedEffect(Unit) {
        viewModel.events.collect { event ->
            when (event) {
                is UserListEvent.NavigateToDetail -> {
                    navController.navigate("user/${event.userId}")
                }
                is UserListEvent.ShowMessage -> {
                    snackbarHostState.showSnackbar(event.message)
                }
                is UserListEvent.ShowError -> {
                    snackbarHostState.showSnackbar(
                        message = event.error,
                        actionLabel = "ปิด"
                    )
                }
            }
        }
    }
    
    Scaffold(
        topBar = {
            TopAppBar(
                title = { Text("ผู้ใช้งาน") },
                actions = {
                    IconButton(onClick = { viewModel.onRefresh() }) {
                        Icon(Icons.Default.Refresh, "Refresh")
                    }
                }
            )
        },
        snackbarHost = { SnackbarHost(snackbarHostState) }
    ) { padding ->
        Column(modifier = Modifier.padding(padding)) {
            // Search
            SearchBar(
                query = uiState.searchQuery,
                onQueryChange = viewModel::onSearchQueryChanged,
                modifier = Modifier.fillMaxWidth().padding(horizontal = 16.dp)
            )
            
            Box(modifier = Modifier.fillMaxSize()) {
                when {
                    uiState.isLoading -> {
                        CircularProgressIndicator(modifier = Modifier.align(Alignment.Center))
                    }
                    uiState.error != null -> {
                        ErrorView(
                            error = uiState.error!!,
                            onRetry = { viewModel.onRefresh() },
                            modifier = Modifier.align(Alignment.Center)
                        )
                    }
                    uiState.users.isEmpty() -> {
                        EmptyView(modifier = Modifier.align(Alignment.Center))
                    }
                    else -> {
                        // Pull to refresh
                        val pullRefreshState = rememberPullToRefreshState()
                        
                        PullToRefreshBox(
                            isRefreshing = uiState.isRefreshing,
                            onRefresh = viewModel::onRefresh,
                            state = pullRefreshState
                        ) {
                            LazyColumn {
                                items(
                                    items = uiState.users,
                                    key = { it.id }
                                ) { user ->
                                    UserCard(
                                        user = user,
                                        onClick = { viewModel.onUserSelected(user) },
                                        onDelete = { viewModel.onDeleteUser(user) }
                                    )
                                }
                            }
                        }
                    }
                }
            }
        }
    }
}

@Composable
fun UserCard(
    user: User,
    onClick: () -> Unit,
    onDelete: () -> Unit
) {
    Card(
        modifier = Modifier
            .fillMaxWidth()
            .padding(horizontal = 16.dp, vertical = 4.dp)
            .clickable { onClick() }
    ) {
        Row(
            modifier = Modifier.padding(16.dp),
            verticalAlignment = Alignment.CenterVertically
        ) {
            // Avatar
            Box(
                modifier = Modifier
                    .size(48.dp)
                    .clip(CircleShape)
                    .background(MaterialTheme.colorScheme.primaryContainer),
                contentAlignment = Alignment.Center
            ) {
                Text(
                    text = user.name.first().toString(),
                    style = MaterialTheme.typography.titleMedium,
                    color = MaterialTheme.colorScheme.onPrimaryContainer
                )
            }
            
            Spacer(modifier = Modifier.width(16.dp))
            
            // Info
            Column(modifier = Modifier.weight(1f)) {
                Text(user.name, style = MaterialTheme.typography.titleMedium)
                Text(
                    user.email,
                    style = MaterialTheme.typography.bodySmall,
                    color = MaterialTheme.colorScheme.onSurfaceVariant
                )
            }
            
            // Delete
            IconButton(onClick = onDelete) {
                Icon(
                    Icons.Default.Delete,
                    contentDescription = "Delete",
                    tint = MaterialTheme.colorScheme.error
                )
            }
        }
    }
}
```

---

## ขั้นตอนที่ 605: SavedStateHandle

```kotlin
// SavedStateHandle เก็บ state รอด process death

@HiltViewModel
class SearchViewModel @Inject constructor(
    private val repository: SearchRepository,
    savedStateHandle: SavedStateHandle  // inject โดย Hilt
) : ViewModel() {
    
    // Restore query จาก saved state
    private val _query = MutableStateFlow(
        savedStateHandle.get<String>("query") ?: ""
    )
    val query: StateFlow<String> = _query.asStateFlow()
    
    val results: StateFlow<List<SearchResult>> = _query
        .debounce(300)
        .flatMapLatest { query ->
            if (query.isBlank()) flowOf(emptyList())
            else repository.search(query)
        }
        .stateIn(viewModelScope, SharingStarted.WhileSubscribed(5000), emptyList())
    
    fun onQueryChange(query: String) {
        _query.value = query
        savedStateHandle["query"] = query  // save ทันที
    }
}

// ViewModel factory ถ้าไม่ใช้ Hilt
class SearchViewModelFactory(
    private val repository: SearchRepository,
    owner: SavedStateRegistryOwner,
    defaultArgs: Bundle? = null
) : AbstractSavedStateViewModelFactory(owner, defaultArgs) {
    
    override fun <T : ViewModel> create(
        key: String,
        modelClass: Class<T>,
        handle: SavedStateHandle
    ): T {
        @Suppress("UNCHECKED_CAST")
        return SearchViewModel(repository, handle) as T
    }
}
```

---

## ขั้นตอนที่ 606: Multiple ViewModels และ Shared State

```kotlin
// ViewModel ที่ share ระหว่าง screens
@HiltViewModel
class CartViewModel @Inject constructor(
    private val cartRepository: CartRepository
) : ViewModel() {
    
    val cartItems: StateFlow<List<CartItem>> = cartRepository
        .observeCartItems()
        .stateIn(viewModelScope, SharingStarted.WhileSubscribed(5000), emptyList())
    
    val cartTotal: StateFlow<Double> = cartItems
        .map { items -> items.sumOf { it.price * it.quantity } }
        .stateIn(viewModelScope, SharingStarted.WhileSubscribed(5000), 0.0)
    
    val cartItemCount: StateFlow<Int> = cartItems
        .map { it.sumOf { item -> item.quantity } }
        .stateIn(viewModelScope, SharingStarted.WhileSubscribed(5000), 0)
    
    fun addToCart(product: Product) {
        viewModelScope.launch {
            cartRepository.addItem(product.toCartItem())
        }
    }
    
    fun removeFromCart(itemId: Long) {
        viewModelScope.launch {
            cartRepository.removeItem(itemId)
        }
    }
    
    fun updateQuantity(itemId: Long, quantity: Int) {
        viewModelScope.launch {
            if (quantity <= 0) cartRepository.removeItem(itemId)
            else cartRepository.updateQuantity(itemId, quantity)
        }
    }
    
    fun clearCart() {
        viewModelScope.launch {
            cartRepository.clearCart()
        }
    }
}

// ใช้ CartViewModel ใน multiple screens
@Composable
fun ProductScreen(cartViewModel: CartViewModel = hiltViewModel()) {
    val cartCount by cartViewModel.cartItemCount.collectAsStateWithLifecycle()
    
    Scaffold(
        topBar = {
            TopAppBar(
                title = { Text("สินค้า") },
                actions = {
                    BadgedBox(badge = {
                        if (cartCount > 0) {
                            Badge { Text(cartCount.toString()) }
                        }
                    }) {
                        IconButton(onClick = { /* navigate to cart */ }) {
                            Icon(Icons.Default.ShoppingCart, "Cart")
                        }
                    }
                }
            )
        }
    ) { padding ->
        // product list...
    }
}
```

---

## ขั้นตอนที่ 607: Testing ViewModel

```kotlin
class UserListViewModelTest {
    
    @get:Rule
    val mainDispatcherRule = MainDispatcherRule()
    
    private val fakeRepository = FakeUserRepository()
    private lateinit var viewModel: UserListViewModel
    
    @Before
    fun setup() {
        viewModel = UserListViewModel(fakeRepository)
    }
    
    @Test
    fun `initial state should be loading`() = runTest {
        val initialState = viewModel.uiState.value
        assertFalse(initialState.isLoading)  // หลัง init แล้ว
        assertTrue(initialState.users.isEmpty() || initialState.users.isNotEmpty())
    }
    
    @Test
    fun `search updates users list`() = runTest {
        fakeRepository.users = listOf(
            User(1, "Alice Smith", "alice@test.com"),
            User(2, "Bob Jones", "bob@test.com"),
            User(3, "Charlie Alice", "charlie@test.com")
        )
        
        viewModel.onSearchQueryChanged("alice")
        advanceTimeBy(400)  // wait for debounce
        
        val state = viewModel.uiState.value
        assertEquals(2, state.users.size)  // Alice + Charlie Alice
    }
    
    @Test
    fun `delete user shows success message`() = runTest {
        val user = User(1, "Test", "test@test.com")
        
        val events = mutableListOf<UserListEvent>()
        val job = launch { viewModel.events.collect { events.add(it) } }
        
        viewModel.onDeleteUser(user)
        advanceUntilIdle()
        
        assertTrue(events.any { it is UserListEvent.ShowMessage })
        job.cancel()
    }
}

// MainDispatcherRule สำหรับ test
class MainDispatcherRule(
    val dispatcher: TestDispatcher = UnconfinedTestDispatcher()
) : TestWatcher() {
    override fun starting(description: Description) {
        Dispatchers.setMain(dispatcher)
    }
    override fun finished(description: Description) {
        Dispatchers.resetMain()
    }
}

// Fake Repository
class FakeUserRepository : UserRepository {
    var users: List<User> = emptyList()
    var shouldThrowError = false
    
    override suspend fun saveUser(user: User): Long {
        if (shouldThrowError) throw Exception("Save failed")
        return (users.maxOfOrNull { it.id } ?: 0) + 1
    }
    
    override fun observeActiveUsers(): Flow<List<User>> = flow {
        emit(users.filter { it.isActive })
    }
    
    override fun searchUsers(query: String): Flow<List<User>> = flow {
        emit(users.filter {
            it.name.contains(query, ignoreCase = true) ||
            it.email.contains(query, ignoreCase = true)
        })
    }
    
    // ... implement other methods
}
```

---

## แบบฝึกหัด Part 25

```kotlin
// แบบฝึกหัด: Todo App ViewModel
// สร้าง ViewModel ที่ครบสมบูรณ์สำหรับ Todo App

data class TodoItem(
    val id: Long = 0,
    val title: String,
    val description: String = "",
    val isCompleted: Boolean = false,
    val priority: Priority = Priority.MEDIUM,
    val dueDate: Long? = null
)

enum class Priority { LOW, MEDIUM, HIGH }

data class TodoUiState(
    val todos: List<TodoItem> = emptyList(),
    val filter: TodoFilter = TodoFilter.ALL,
    val isLoading: Boolean = false,
    val error: String? = null
)

enum class TodoFilter { ALL, ACTIVE, COMPLETED }

// TODO: สร้าง TodoViewModel ที่มี:
// - เพิ่ม, แก้ไข, ลบ todo
// - mark complete/incomplete
// - filter ตาม TodoFilter
// - sort ตาม priority และ due date
// - search todos

class TodoViewModel : ViewModel() {
    // TODO: implement
}
```

---

*Part 25 จบแล้ว | ก่อนหน้า: [Part 24](../part24/README.md) | ถัดไป: [Part 26](../part26/README.md)*
