# Part 61: Advanced Testing Strategies
## ขั้นตอนที่ 1001-1025

---

## ขั้นตอนที่ 1001: Test Pyramid

```
Test Pyramid:

                /\
               /  \
              / E2E\        ← น้อย, ช้า, แพง
             /──────\
            / Integr-\     ← ปานกลาง
           / ation    \
          /────────────\
         /  Unit Tests  \  ← มาก, เร็ว, ถูก
        /________________\

Android Test Pyramid:
- Unit Tests (70%): ViewModel, UseCase, Repository logic
- Integration Tests (20%): Room, Retrofit, Hilt component
- UI/E2E Tests (10%): Compose UI, Navigation flows

สิ่งที่ต้องทดสอบ:
1. Business logic (pure Kotlin)
2. Data layer (Room in-memory)
3. ViewModel state transitions
4. Compose UI interactions
5. Navigation flows
```

---

## ขั้นตอนที่ 1002: Test Doubles แบบละเอียด

```kotlin
// ============================================
// Fake vs Mock vs Stub vs Spy
// ============================================

// STUB - คืนค่าที่กำหนดไว้ล่วงหน้า
class StubUserRepository : UserRepository {
    override suspend fun getUser(id: Long): User {
        return User(id, "Test User", "test@example.com")  // hardcoded
    }
    
    override suspend fun updateUser(user: User): User = user
    override fun observeUsers(): Flow<List<User>> = flowOf(emptyList())
}

// FAKE - implementation ที่ทำงานได้จริงแต่ simplified
class FakeUserRepository : UserRepository {
    private val users = mutableMapOf<Long, User>()
    private var nextId = 1L
    
    override suspend fun getUser(id: Long): User {
        return users[id] ?: throw NotFoundException("User $id not found")
    }
    
    override suspend fun updateUser(user: User): User {
        users[user.id] = user
        return user
    }
    
    override fun observeUsers(): Flow<List<User>> {
        return flow {
            emit(users.values.toList())
        }
    }
    
    fun addUser(user: User) { users[user.id] = user }
    fun clearAll() { users.clear() }
}

// MOCK - verify ว่า method ถูกเรียกหรือไม่ (ด้วย MockK)
class MockTest {
    val mockRepository = mockk<UserRepository>()
    
    @Test
    fun `should call getUser with correct id`() = runTest {
        coEvery { mockRepository.getUser(any()) } returns User(1, "Alice", "alice@ex.com")
        
        val viewModel = UserViewModel(mockRepository)
        viewModel.loadUser(1L)
        
        coVerify(exactly = 1) { mockRepository.getUser(1L) }
    }
}

// SPY - partial mock (เรียก real implementation แต่ verify ได้)
class SpyTest {
    val realRepository = FakeUserRepository()
    val spyRepository = spyk(realRepository)
    
    @Test
    fun `tracks calls to real implementation`() = runTest {
        spyRepository.addUser(User(1, "Alice", "alice@ex.com"))
        
        val user = spyRepository.getUser(1L)
        
        assertEquals("Alice", user.name)
        coVerify { spyRepository.getUser(1L) }
    }
}
```

---

## ขั้นตอนที่ 1003: Integration Testing กับ Room

```kotlin
// ============================================
// Room Integration Test
// ============================================

@RunWith(AndroidJUnit4::class)
class UserDaoIntegrationTest {
    
    private lateinit var database: AppDatabase
    private lateinit var userDao: UserDao
    
    @Before
    fun setup() {
        database = Room.inMemoryDatabaseBuilder(
            ApplicationProvider.getApplicationContext(),
            AppDatabase::class.java
        )
            .allowMainThreadQueries()  // เฉพาะ test เท่านั้น
            .build()
        
        userDao = database.userDao()
    }
    
    @After
    fun tearDown() {
        database.close()
    }
    
    @Test
    fun insertAndRetrieveUser() = runBlocking {
        val user = UserEntity(id = 1, name = "Alice", email = "alice@ex.com")
        userDao.insert(user)
        
        val retrieved = userDao.findById(1)
        
        assertNotNull(retrieved)
        assertEquals("Alice", retrieved?.name)
    }
    
    @Test
    fun observeUsers_emitsUpdates() = runBlocking {
        val results = mutableListOf<List<UserEntity>>()
        
        val job = launch {
            userDao.observeAll().take(2).collect { results.add(it) }
        }
        
        userDao.insert(UserEntity(1, "Alice", "alice@ex.com"))
        delay(100)
        
        job.cancel()
        
        assertEquals(2, results.size)
        assertEquals(0, results[0].size)  // initial empty
        assertEquals(1, results[1].size)  // after insert
    }
    
    @Test
    fun deleteUser_removesFromDb() = runBlocking {
        val user = UserEntity(1, "Alice", "alice@ex.com")
        userDao.insert(user)
        
        userDao.delete(user)
        
        val retrieved = userDao.findById(1)
        assertNull(retrieved)
    }
}
```

---

## ขั้นตอนที่ 1004: Compose UI Testing ขั้นสูง

```kotlin
// ============================================
// Compose UI Tests
// ============================================

@RunWith(AndroidJUnit4::class)
class LoginScreenTest {
    
    @get:Rule
    val composeTestRule = createComposeRule()
    
    private val fakeAuthRepository = FakeAuthRepository()
    
    @Before
    fun setup() {
        composeTestRule.setContent {
            AppTheme {
                LoginScreen(
                    viewModel = LoginViewModel(fakeAuthRepository)
                )
            }
        }
    }
    
    @Test
    fun loginButton_disabled_when_fields_empty() {
        composeTestRule
            .onNodeWithText("เข้าสู่ระบบ")
            .assertIsNotEnabled()
    }
    
    @Test
    fun loginButton_enabled_when_fields_filled() {
        composeTestRule
            .onNodeWithTag("email_field")
            .performTextInput("test@example.com")
        
        composeTestRule
            .onNodeWithTag("password_field")
            .performTextInput("password123")
        
        composeTestRule
            .onNodeWithText("เข้าสู่ระบบ")
            .assertIsEnabled()
    }
    
    @Test
    fun shows_error_on_invalid_credentials() {
        fakeAuthRepository.shouldThrow = AuthException.InvalidCredentials
        
        composeTestRule
            .onNodeWithTag("email_field")
            .performTextInput("wrong@example.com")
        
        composeTestRule
            .onNodeWithTag("password_field")
            .performTextInput("wrongpass")
        
        composeTestRule
            .onNodeWithText("เข้าสู่ระบบ")
            .performClick()
        
        composeTestRule.waitUntil {
            composeTestRule
                .onAllNodesWithTag("error_message")
                .fetchSemanticsNodes().isNotEmpty()
        }
        
        composeTestRule
            .onNodeWithTag("error_message")
            .assertTextContains("อีเมลหรือรหัสผ่านไม่ถูกต้อง")
    }
    
    @Test
    fun screenshot_matches_baseline() {
        // Snapshot testing (ต้องใช้ library เช่น Paparazzi)
        // paparazziRule.snapshot { LoginScreen(...) }
    }
}

// Semantic helpers
fun SemanticsNodeInteractionsProvider.onNodeWithTag(tag: String) =
    onNode(hasTestTag(tag))
```

---

## ขั้นตอนที่ 1005: Property-Based Testing

```kotlin
// ============================================
// Property-Based Testing กับ Kotest
// ============================================

// build.gradle.kts
// testImplementation("io.kotest:kotest-runner-junit5:5.x")
// testImplementation("io.kotest:kotest-property:5.x")

class UserValidatorPropertyTest : FreeSpec({
    
    "email validation" - {
        "valid emails should pass" {
            checkAll<String> { domain ->
                val email = "user@${domain.filter { it.isLetterOrDigit() }}.com"
                if (email.matches(Regex("[a-zA-Z0-9]+@[a-zA-Z0-9]+\\.com"))) {
                    UserValidator.isValidEmail(email) shouldBe true
                }
            }
        }
        
        "emails without @ should fail" {
            forAll(
                row("notanemail"),
                row("missing_at.com"),
                row("also_missing"),
                row("")
            ) { email ->
                UserValidator.isValidEmail(email) shouldBe false
            }
        }
    }
    
    "name length" - {
        "names 2-50 chars should be valid" {
            checkAll(Arb.string(2..50)) { name ->
                UserValidator.isValidName(name) shouldBe true
            }
        }
    }
})

// Simple property test tanpa library
@Test
fun `addition is commutative`() {
    repeat(100) {
        val a = Random.nextInt(-1000, 1000)
        val b = Random.nextInt(-1000, 1000)
        assertEquals(a + b, b + a)
    }
}
```

---

## แบบฝึกหัด Part 61

```kotlin
// แบบฝึกหัด: Test-Driven Development (TDD)

// ทำตาม Red → Green → Refactor cycle:

// Step 1: RED - เขียน test ที่ fail ก่อน
@Test
fun `calculate discount returns 10% for premium users`() {
    val calculator = DiscountCalculator()
    val user = User(id = 1, isPremium = true)
    val price = 1000.0
    
    val discount = calculator.calculate(user, price)
    
    assertEquals(100.0, discount, 0.01)
}

// Step 2: GREEN - เขียน code ให้ผ่าน
class DiscountCalculator {
    fun calculate(user: User, price: Double): Double {
        // TODO: implement
        return 0.0  // ← test จะ fail
    }
}

// Step 3: REFACTOR - ปรับปรุง code
// TODO: implement calculate() ให้ test ผ่าน
// แล้วเพิ่ม tests:
// - 20% for premium + first purchase
// - 0% for non-premium
// - minimum discount = 0
// - maximum discount = 50%
```

---

*Part 61 จบแล้ว | ก่อนหน้า: [Part 60](../part60/README.md) | ถัดไป: [Part 62](../part62/README.md)*
