# Part 29: Testing ใน Android
## ขั้นตอนที่ 701-725

---

## ขั้นตอนที่ 701: Testing Pyramid

```
        /\
       /  \
      / UI \           UI Tests (Instrumented)
     /  Tests\         - Espresso, Compose Testing
    /----------\       ช้า, ต้องใช้ Device/Emulator
   / Integration\      
  /    Tests    \      Integration Tests
 /              \      - Room, Repository tests
/-----------------\    เร็วปานกลาง
/                  \   
/    Unit Tests     \  Unit Tests (Local)
/--------------------\ - ViewModel, UseCase tests
                       เร็วมาก, ทำงานบน JVM
```

---

## ขั้นตอนที่ 702: Unit Tests

```kotlin
// build.gradle.kts (test dependencies)
// testImplementation("junit:junit:4.13.2")
// testImplementation("org.mockito.kotlin:mockito-kotlin:5.3.x")
// testImplementation("io.mockk:mockk:1.13.x")
// testImplementation("org.jetbrains.kotlinx:kotlinx-coroutines-test:1.8.x")
// testImplementation("app.cash.turbine:turbine:1.1.x")  // Flow testing

// ============================================
// JUnit 4 + MockK
// ============================================

class UserRepositoryTest {
    
    // MockK mocks
    private val userDao = mockk<UserDao>()
    private val apiService = mockk<ApiService>()
    
    private lateinit var repository: UserRepositoryImpl
    
    @Before
    fun setup() {
        repository = UserRepositoryImpl(userDao, apiService, Dispatchers.IO)
    }
    
    @Test
    fun `getUsers returns mapped domain objects`() = runTest {
        // Arrange
        val entities = listOf(
            UserEntity(1, "Alice", "alice@test.com"),
            UserEntity(2, "Bob", "bob@test.com")
        )
        coEvery { userDao.getAllUsers() } returns entities
        
        // Act
        val result = repository.getUsers()
        
        // Assert
        assertEquals(2, result.size)
        assertEquals("Alice", result[0].name)
        assertEquals("alice@test.com", result[0].email)
        
        // Verify interaction
        coVerify(exactly = 1) { userDao.getAllUsers() }
    }
    
    @Test
    fun `getUser returns null when not found`() = runTest {
        coEvery { userDao.getUserById(999) } returns null
        
        val result = repository.getUser(999)
        
        assertNull(result)
    }
    
    @Test
    fun `saveUser returns new id`() = runTest {
        val user = User(name = "Charlie", email = "charlie@test.com")
        coEvery { userDao.insertUser(any()) } returns 42L
        
        val id = repository.saveUser(user)
        
        assertEquals(42L, id)
        coVerify { userDao.insertUser(match { it.name == "Charlie" }) }
    }
}
```

---

## ขั้นตอนที่ 703: Testing ViewModel

```kotlin
class UserViewModelTest {
    
    @get:Rule
    val mainDispatcherRule = MainDispatcherRule()
    
    private val fakeRepository = FakeUserRepository()
    private lateinit var viewModel: UserListViewModel
    
    @Before
    fun setup() {
        viewModel = UserListViewModel(fakeRepository)
    }
    
    // ============================================
    // Test StateFlow
    // ============================================
    
    @Test
    fun `initial state loads users`() = runTest {
        fakeRepository.setUsers(listOf(
            User(1, "Alice", "alice@test.com"),
            User(2, "Bob", "bob@test.com")
        ))
        
        // Re-create ViewModel after setting data
        viewModel = UserListViewModel(fakeRepository)
        advanceUntilIdle()
        
        assertEquals(2, viewModel.uiState.value.users.size)
        assertFalse(viewModel.uiState.value.isLoading)
    }
    
    @Test
    fun `search filters users`() = runTest {
        fakeRepository.setUsers(listOf(
            User(1, "Alice Smith", "alice@test.com"),
            User(2, "Bob Jones", "bob@test.com"),
            User(3, "Alice Jones", "alice2@test.com")
        ))
        viewModel = UserListViewModel(fakeRepository)
        
        viewModel.onSearchQueryChanged("alice")
        advanceTimeBy(400)  // debounce delay
        
        val users = viewModel.uiState.value.users
        assertEquals(2, users.size)
        assertTrue(users.all { it.name.contains("Alice", ignoreCase = true) })
    }
    
    // ============================================
    // Test Flow Events ด้วย Turbine
    // ============================================
    
    @Test
    fun `delete user emits success event`() = runTest {
        val user = User(1, "Test", "test@test.com")
        
        viewModel.events.test {
            viewModel.onDeleteUser(user)
            
            val event = awaitItem()
            assertTrue(event is UserListEvent.ShowMessage)
            
            cancelAndIgnoreRemainingEvents()
        }
    }
    
    @Test
    fun `delete user failure emits error event`() = runTest {
        fakeRepository.shouldThrowOnDelete = true
        val user = User(1, "Test", "test@test.com")
        
        viewModel.events.test {
            viewModel.onDeleteUser(user)
            
            val event = awaitItem()
            assertTrue(event is UserListEvent.ShowError)
            
            cancelAndIgnoreRemainingEvents()
        }
    }
}

// ============================================
// MainDispatcherRule
// ============================================

class MainDispatcherRule(
    private val dispatcher: TestDispatcher = UnconfinedTestDispatcher()
) : TestWatcher() {
    override fun starting(description: Description) {
        Dispatchers.setMain(dispatcher)
    }
    override fun finished(description: Description) {
        Dispatchers.resetMain()
    }
}
```

---

## ขั้นตอนที่ 704: Testing Room Database

```kotlin
// ============================================
// Room Database Test (Instrumented)
// ============================================

@RunWith(AndroidJUnit4::class)
class UserDaoTest {
    
    private lateinit var database: AppDatabase
    private lateinit var userDao: UserDao
    
    @Before
    fun setup() {
        database = Room.inMemoryDatabaseBuilder(
            ApplicationProvider.getApplicationContext(),
            AppDatabase::class.java
        )
        .allowMainThreadQueries()
        .build()
        
        userDao = database.userDao()
    }
    
    @After
    fun teardown() {
        database.close()
    }
    
    @Test
    fun insertAndGetUser() = runTest {
        val user = UserEntity(name = "Alice", email = "alice@test.com")
        val id = userDao.insertUser(user)
        
        val retrieved = userDao.getUserById(id)
        
        assertNotNull(retrieved)
        assertEquals("Alice", retrieved!!.name)
        assertEquals("alice@test.com", retrieved.email)
    }
    
    @Test
    fun deleteUser() = runTest {
        val user = UserEntity(name = "Bob", email = "bob@test.com")
        val id = userDao.insertUser(user)
        
        userDao.deleteUserById(id)
        
        val retrieved = userDao.getUserById(id)
        assertNull(retrieved)
    }
    
    @Test
    fun searchUsers() = runTest {
        userDao.insertUsers(listOf(
            UserEntity(name = "Alice Smith", email = "alice@test.com"),
            UserEntity(name = "Bob Jones", email = "bob@test.com"),
            UserEntity(name = "Charlie", email = "charlie.alice@test.com")
        ))
        
        val result = userDao.searchUsers("alice").first()
        
        assertEquals(2, result.size)  // Alice Smith + charlie.alice
    }
    
    @Test
    fun observeUsers_emitsUpdates() = runTest {
        userDao.observeActiveUsers().test {
            // Initial empty state
            assertEquals(0, awaitItem().size)
            
            // Insert
            userDao.insertUser(UserEntity(name = "Alice", email = "alice@test.com"))
            assertEquals(1, awaitItem().size)
            
            // Insert another
            userDao.insertUser(UserEntity(name = "Bob", email = "bob@test.com"))
            assertEquals(2, awaitItem().size)
            
            cancelAndIgnoreRemainingEvents()
        }
    }
}
```

---

## ขั้นตอนที่ 705: Compose UI Testing

```kotlin
// androidTest dependencies
// androidTestImplementation("androidx.compose.ui:ui-test-junit4")
// debugImplementation("androidx.compose.ui:ui-test-manifest")

@RunWith(AndroidJUnit4::class)
class UserListScreenTest {
    
    @get:Rule
    val composeRule = createComposeRule()
    
    @Test
    fun userList_displaysUsers() {
        val users = listOf(
            User(1, "Alice", "alice@test.com"),
            User(2, "Bob", "bob@test.com")
        )
        
        composeRule.setContent {
            UserListContent(
                users = users,
                onUserClick = {},
                onDeleteUser = {}
            )
        }
        
        composeRule.onNodeWithText("Alice").assertIsDisplayed()
        composeRule.onNodeWithText("bob@test.com").assertIsDisplayed()
    }
    
    @Test
    fun userList_clickUser_triggersCallback() {
        var clickedUser: User? = null
        val users = listOf(User(1, "Alice", "alice@test.com"))
        
        composeRule.setContent {
            UserListContent(
                users = users,
                onUserClick = { clickedUser = it },
                onDeleteUser = {}
            )
        }
        
        composeRule.onNodeWithText("Alice").performClick()
        
        assertNotNull(clickedUser)
        assertEquals(1L, clickedUser!!.id)
    }
    
    @Test
    fun searchBar_filtersUsers() {
        composeRule.setContent {
            UserListScreen()  // with real ViewModel
        }
        
        // Type in search
        composeRule
            .onNodeWithTag("search_field")
            .performTextInput("alice")
        
        // Wait for debounce
        composeRule.mainClock.advanceTimeBy(400)
        
        composeRule.onNodeWithText("Alice").assertIsDisplayed()
        composeRule.onNodeWithText("Bob").assertDoesNotExist()
    }
    
    @Test
    fun loadingState_showsProgressIndicator() {
        composeRule.setContent {
            UserListContent(
                users = emptyList(),
                isLoading = true,
                onUserClick = {},
                onDeleteUser = {}
            )
        }
        
        composeRule.onNodeWithTag("loading_indicator").assertIsDisplayed()
    }
    
    @Test
    fun errorState_showsErrorMessage() {
        composeRule.setContent {
            UserListContent(
                users = emptyList(),
                error = "Network error",
                onUserClick = {},
                onDeleteUser = {}
            )
        }
        
        composeRule.onNodeWithText("Network error").assertIsDisplayed()
        composeRule.onNodeWithText("ลองใหม่").assertIsDisplayed()
    }
}
```

---

## ขั้นตอนที่ 706: Test Doubles

```kotlin
// ============================================
// Types of Test Doubles
// ============================================

// 1. Fake - implementation จริงแต่ simplified
class FakeUserRepository : UserRepository {
    private val _users = MutableStateFlow<List<User>>(emptyList())
    
    var shouldThrowOnSave = false
    var shouldThrowOnDelete = false
    
    fun setUsers(users: List<User>) { _users.value = users }
    
    override suspend fun saveUser(user: User): Long {
        if (shouldThrowOnSave) throw Exception("Save error")
        val id = (_users.value.maxOfOrNull { it.id } ?: 0) + 1
        _users.value = _users.value + user.copy(id = id)
        return id
    }
    
    override suspend fun deleteUser(id: Long) {
        if (shouldThrowOnDelete) throw Exception("Delete error")
        _users.value = _users.value.filter { it.id != id }
    }
    
    override fun observeActiveUsers(): Flow<List<User>> =
        _users.map { it.filter { u -> u.isActive } }
    
    override fun searchUsers(query: String): Flow<List<User>> =
        _users.map { users ->
            users.filter {
                it.name.contains(query, true) || it.email.contains(query, true)
            }
        }
    
    override suspend fun getUser(id: Long): User? = _users.value.find { it.id == id }
    override suspend fun getAllUsers(): List<User> = _users.value
    override suspend fun updateUser(user: User) {
        _users.value = _users.value.map { if (it.id == user.id) user else it }
    }
}

// 2. Mock ด้วย MockK
class UserServiceTest {
    
    private val repository = mockk<UserRepository>()
    private val service = UserService(repository)
    
    @Test
    fun `activate user calls repository`() = runTest {
        val user = User(1, "Alice", "alice@test.com", isActive = false)
        
        coEvery { repository.getUser(1) } returns user
        coEvery { repository.updateUser(any()) } returns Unit
        
        service.activateUser(1)
        
        coVerify {
            repository.updateUser(match { it.id == 1L && it.isActive == true })
        }
    }
}

// 3. Spy - wraps real object ด้วย ability to verify
class SpyTest {
    
    @Test
    fun `spy test`() = runTest {
        val realRepository = FakeUserRepository()
        val spyRepository = spyk(realRepository)
        
        spyRepository.saveUser(User(name = "Test", email = "test@test.com"))
        
        coVerify { spyRepository.saveUser(any()) }
    }
}
```

---

## แบบฝึกหัด Part 29

```kotlin
// แบบฝึกหัด: เขียน Tests สำหรับ Calculator App

class Calculator {
    fun add(a: Double, b: Double): Double = a + b
    fun subtract(a: Double, b: Double): Double = a - b
    fun multiply(a: Double, b: Double): Double = a * b
    fun divide(a: Double, b: Double): Double {
        if (b == 0.0) throw ArithmeticException("Division by zero")
        return a / b
    }
    fun percentage(value: Double, percent: Double): Double = value * percent / 100
}

// TODO: เขียน CalculatorTest ที่ครอบคลุม:
// 1. Test ทุก operation ด้วย normal values
// 2. Test edge cases (0, negative numbers, very large numbers)
// 3. Test exception: divide by zero
// 4. Test percentage calculation

// CalculatorViewModel
class CalculatorViewModel(private val calculator: Calculator) : ViewModel() {
    private val _display = MutableStateFlow("0")
    val display: StateFlow<String> = _display.asStateFlow()
    
    private var currentInput = ""
    private var previousInput = ""
    private var pendingOperation: String? = null
    
    fun onDigit(digit: String) { /* TODO */ }
    fun onOperation(op: String) { /* TODO */ }
    fun onEquals() { /* TODO */ }
    fun onClear() { /* TODO */ }
}

// TODO: เขียน CalculatorViewModelTest
```

---

*Part 29 จบแล้ว | ก่อนหน้า: [Part 28](../part28/README.md) | ถัดไป: [Part 30](../part30/README.md)*
