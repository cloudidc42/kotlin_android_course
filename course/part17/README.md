# Part 17: Testing ใน Kotlin
## ขั้นตอนที่ 391-420

---

## ขั้นตอนที่ 391: JUnit และ Kotlin Test

```kotlin
// build.gradle.kts
// testImplementation("junit:junit:4.13.2")
// testImplementation("org.jetbrains.kotlin:kotlin-test")
// testImplementation("io.mockk:mockk:1.13.x")
// testImplementation("org.jetbrains.kotlinx:kotlinx-coroutines-test:1.8.x")
// testImplementation("app.cash.turbine:turbine:1.1.x")

// ============================================
// JUnit 4 Basics
// ============================================

import org.junit.Test
import org.junit.Before
import org.junit.After
import org.junit.Assert.*

class MathUtilsTest {
    
    private lateinit var calculator: Calculator
    
    @Before
    fun setUp() {
        calculator = Calculator()
    }
    
    @After
    fun tearDown() {
        // cleanup
    }
    
    @Test
    fun `add two positive numbers`() {
        val result = calculator.add(2.0, 3.0)
        assertEquals(5.0, result, 0.001)
    }
    
    @Test
    fun `add negative numbers`() {
        val result = calculator.add(-1.0, -2.0)
        assertEquals(-3.0, result, 0.001)
    }
    
    @Test(expected = ArithmeticException::class)
    fun `divide by zero throws exception`() {
        calculator.divide(10.0, 0.0)
    }
    
    @Test
    fun `divide by zero with assertThrows`() {
        val exception = assertThrows(ArithmeticException::class.java) {
            calculator.divide(10.0, 0.0)
        }
        assertEquals("Division by zero", exception.message)
    }
}

// Kotlin Test (สะดวกกว่า)
import kotlin.test.*

class StringExtensionsTest {
    
    @Test
    fun `isPalindrome returns true for palindrome`() {
        assertTrue("racecar".isPalindrome())
        assertTrue("level".isPalindrome())
        assertTrue("a".isPalindrome())
        assertTrue("".isPalindrome())
    }
    
    @Test
    fun `isPalindrome returns false for non-palindrome`() {
        assertFalse("hello".isPalindrome())
        assertFalse("kotlin".isPalindrome())
    }
    
    @Test
    fun `isEmail validates email format`() {
        assertTrue("user@example.com".isEmail())
        assertTrue("test.email@domain.co.th".isEmail())
        assertFalse("not-an-email".isEmail())
        assertFalse("@missing-user.com".isEmail())
        assertFalse("missing-domain@".isEmail())
    }
}
```

---

## ขั้นตอนที่ 392: Parameterized Tests

```kotlin
import org.junit.runner.RunWith
import org.junit.runners.Parameterized

@RunWith(Parameterized::class)
class AdditionTest(
    private val a: Double,
    private val b: Double,
    private val expected: Double
) {
    companion object {
        @JvmStatic
        @Parameterized.Parameters(name = "{0} + {1} = {2}")
        fun data() = listOf(
            arrayOf(1.0, 2.0, 3.0),
            arrayOf(-1.0, 1.0, 0.0),
            arrayOf(0.0, 0.0, 0.0),
            arrayOf(100.0, -50.0, 50.0),
            arrayOf(Double.MAX_VALUE, 0.0, Double.MAX_VALUE)
        )
    }
    
    @Test
    fun testAdd() {
        val calc = Calculator()
        assertEquals(expected, calc.add(a, b), 0.0001)
    }
}

// JUnit 5 Parameterized (ถ้าใช้ JUnit 5)
// @ParameterizedTest
// @ValueSource(strings = ["racecar", "level", "madam", "noon"])
// fun isPalindrome_validPalindromes_returnsTrue(word: String) {
//     assertTrue(word.isPalindrome())
// }
```

---

## ขั้นตอนที่ 393: MockK

```kotlin
import io.mockk.*

interface UserService {
    suspend fun getUser(id: Long): User?
    suspend fun saveUser(user: User): Long
    fun observeUsers(): Flow<List<User>>
}

class UserManagerTest {
    
    private val userService = mockk<UserService>()
    private val userManager = UserManager(userService)
    
    @Before
    fun setup() {
        // Default behavior
        coEvery { userService.getUser(any()) } returns null
    }
    
    // ============================================
    // coEvery - mock suspend functions
    // ============================================
    
    @Test
    fun `getUserName returns name when user found`() = runTest {
        coEvery { userService.getUser(1) } returns User(1, "Alice", "alice@test.com")
        
        val name = userManager.getUserName(1)
        
        assertEquals("Alice", name)
    }
    
    @Test
    fun `getUserName returns null when user not found`() = runTest {
        coEvery { userService.getUser(999) } returns null
        
        val name = userManager.getUserName(999)
        
        assertNull(name)
    }
    
    // ============================================
    // every - mock regular functions
    // ============================================
    
    @Test
    fun `observeUsers returns flow from service`() = runTest {
        val users = listOf(User(1, "Alice", "alice@test.com"))
        every { userService.observeUsers() } returns flowOf(users)
        
        val observed = userManager.getActiveUsers().first()
        
        assertEquals(users, observed)
    }
    
    // ============================================
    // Verification
    // ============================================
    
    @Test
    fun `saveUser calls service once`() = runTest {
        val user = User(name = "Bob", email = "bob@test.com")
        coEvery { userService.saveUser(any()) } returns 42L
        
        userManager.createUser("Bob", "bob@test.com")
        
        coVerify(exactly = 1) { userService.saveUser(any()) }
        coVerify { userService.saveUser(match { it.name == "Bob" }) }
    }
    
    @Test
    fun `no unnecessary service calls`() = runTest {
        userManager.doNothing()
        
        coVerify(exactly = 0) { userService.getUser(any()) }
        verify(exactly = 0) { userService.observeUsers() }
    }
    
    // ============================================
    // Capturing arguments
    // ============================================
    
    @Test
    fun `saveUser called with correct data`() = runTest {
        val capturedUser = slot<User>()
        coEvery { userService.saveUser(capture(capturedUser)) } returns 1L
        
        userManager.createUser("Charlie", "charlie@test.com")
        
        assertEquals("Charlie", capturedUser.captured.name)
        assertEquals("charlie@test.com", capturedUser.captured.email)
        assertTrue(capturedUser.captured.isActive)
    }
    
    // ============================================
    // Exception behavior
    // ============================================
    
    @Test
    fun `handles service exception`() = runTest {
        coEvery { userService.getUser(any()) } throws Exception("Network error")
        
        val result = userManager.getUserSafely(1)
        
        assertTrue(result is Result.Error)
    }
}
```

---

## ขั้นตอนที่ 394: Testing Flows ด้วย Turbine

```kotlin
import app.cash.turbine.test

class UserFlowTest {
    
    private val fakeRepository = FakeUserRepository()
    
    @Test
    fun `observe users emits updates`() = runTest {
        fakeRepository.observeUsers().test {
            // ตรวจสอบ initial emission
            assertEquals(emptyList(), awaitItem())
            
            // เพิ่ม user
            fakeRepository.addUser(User(1, "Alice", "alice@test.com"))
            
            // ตรวจสอบ update
            val updated = awaitItem()
            assertEquals(1, updated.size)
            assertEquals("Alice", updated[0].name)
            
            cancelAndIgnoreRemainingEvents()
        }
    }
    
    @Test
    fun `search flow debounces and filters`() = runTest {
        fakeRepository.addUsers(listOf(
            User(1, "Alice Smith", "alice@test.com"),
            User(2, "Bob Jones", "bob@test.com")
        ))
        
        val searchQuery = MutableStateFlow("")
        
        val results = searchQuery
            .debounce(300)
            .flatMapLatest { q ->
                if (q.isBlank()) fakeRepository.observeUsers()
                else fakeRepository.searchUsers(q)
            }
        
        results.test {
            assertEquals(2, awaitItem().size)  // all users
            
            searchQuery.value = "alice"
            advanceTimeBy(400)  // past debounce
            
            val filtered = awaitItem()
            assertEquals(1, filtered.size)
            assertEquals("Alice Smith", filtered[0].name)
            
            cancelAndIgnoreRemainingEvents()
        }
    }
    
    @Test
    fun `flow catches and emits error`() = runTest {
        fakeRepository.shouldThrowError = true
        
        fakeRepository.observeUsersWithError()
            .catch { e ->
                emit(emptyList())
            }
            .test {
                assertEquals(emptyList<User>(), awaitItem())
                awaitComplete()
            }
    }
}
```

---

## ขั้นตอนที่ 395: Test-Driven Development (TDD)

```kotlin
// TDD Cycle: Red → Green → Refactor

// ============================================
// Step 1: Write failing test (RED)
// ============================================

class PasswordValidatorTest {
    
    private val validator = PasswordValidator()
    
    @Test
    fun `valid password passes all rules`() {
        val result = validator.validate("Secure@123")
        assertTrue(result.isValid)
        assertTrue(result.errors.isEmpty())
    }
    
    @Test
    fun `too short password fails`() {
        val result = validator.validate("Ab1@")
        assertFalse(result.isValid)
        assertTrue(result.errors.contains(PasswordError.TOO_SHORT))
    }
    
    @Test
    fun `no uppercase fails`() {
        val result = validator.validate("secure@123")
        assertFalse(result.isValid)
        assertTrue(result.errors.contains(PasswordError.NO_UPPERCASE))
    }
    
    @Test
    fun `no number fails`() {
        val result = validator.validate("Secure@pass")
        assertFalse(result.isValid)
        assertTrue(result.errors.contains(PasswordError.NO_NUMBER))
    }
    
    @Test
    fun `no special char fails`() {
        val result = validator.validate("SecurePass123")
        assertFalse(result.isValid)
        assertTrue(result.errors.contains(PasswordError.NO_SPECIAL_CHAR))
    }
    
    @Test
    fun `multiple errors are reported`() {
        val result = validator.validate("short")  // short + no uppercase + no number + no special
        assertFalse(result.isValid)
        assertEquals(4, result.errors.size)
    }
}

// ============================================
// Step 2: Write minimal code to pass (GREEN)
// ============================================

enum class PasswordError {
    TOO_SHORT, NO_UPPERCASE, NO_LOWERCASE, NO_NUMBER, NO_SPECIAL_CHAR
}

data class ValidationResult(
    val isValid: Boolean,
    val errors: List<PasswordError>
)

class PasswordValidator {
    private val minLength = 8
    private val specialChars = "!@#$%^&*()_+-=[]{}|;':\",./<>?"
    
    fun validate(password: String): ValidationResult {
        val errors = mutableListOf<PasswordError>()
        
        if (password.length < minLength) errors.add(PasswordError.TOO_SHORT)
        if (!password.any { it.isUpperCase() }) errors.add(PasswordError.NO_UPPERCASE)
        if (!password.any { it.isLowerCase() }) errors.add(PasswordError.NO_LOWERCASE)
        if (!password.any { it.isDigit() }) errors.add(PasswordError.NO_NUMBER)
        if (!password.any { it in specialChars }) errors.add(PasswordError.NO_SPECIAL_CHAR)
        
        return ValidationResult(errors.isEmpty(), errors)
    }
}

// ============================================
// Step 3: Refactor (REFACTOR)
// ============================================

class PasswordValidatorRefactored {
    
    data class Rule(
        val error: PasswordError,
        val check: (String) -> Boolean
    )
    
    private val rules = listOf(
        Rule(PasswordError.TOO_SHORT) { it.length >= 8 },
        Rule(PasswordError.NO_UPPERCASE) { it.any(Char::isUpperCase) },
        Rule(PasswordError.NO_LOWERCASE) { it.any(Char::isLowerCase) },
        Rule(PasswordError.NO_NUMBER) { it.any(Char::isDigit) },
        Rule(PasswordError.NO_SPECIAL_CHAR) { pwd ->
            pwd.any { it in "!@#$%^&*()_+-=[]{}|;':\",./<>?" }
        }
    )
    
    fun validate(password: String): ValidationResult {
        val errors = rules.filterNot { it.check(password) }.map { it.error }
        return ValidationResult(errors.isEmpty(), errors)
    }
}
```

---

## แบบฝึกหัด Part 17

```kotlin
// แบบฝึกหัด: เขียน Tests ด้วย TDD สำหรับ BankAccount

class BankAccount(val owner: String, initialBalance: Double = 0.0) {
    private var _balance: Double = initialBalance
    val balance: Double get() = _balance
    private val _transactions = mutableListOf<Transaction>()
    val transactions: List<Transaction> get() = _transactions.toList()
    
    // TODO: implement
    fun deposit(amount: Double) { /* TODO */ }
    fun withdraw(amount: Double) { /* TODO */ }
    fun transfer(target: BankAccount, amount: Double) { /* TODO */ }
}

data class Transaction(
    val type: String,  // "deposit", "withdrawal", "transfer_in", "transfer_out"
    val amount: Double,
    val timestamp: Long = System.currentTimeMillis()
)

// TODO: เขียน Tests ก่อน implement:
// 1. deposit บวกเพิ่ม balance
// 2. withdraw ลด balance
// 3. withdraw เมื่อ balance ไม่พอต้อง throw InsufficientFundsException
// 4. deposit/withdraw ที่ amount <= 0 ต้อง throw IllegalArgumentException
// 5. transfer โอนเงินระหว่าง accounts
// 6. transactions บันทึกประวัติถูกต้อง
```

---

*Part 17 จบแล้ว | ก่อนหน้า: [Part 16](../part16/README.md) | ถัดไป: [Part 18](../part18/README.md)*
