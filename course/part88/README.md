# Part 88: Advanced Testing Strategies
## ขั้นตอนที่ 1676-1700

---

## ขั้นตอนที่ 1676: Test Double Strategy

```
Test Doubles:

1. Dummy - ส่งเข้า constructor แต่ไม่ถูกใช้
   - OrderPrinter(logger = DummyLogger())

2. Stub - return hardcoded values
   - userDao.getUser(1) returns fakeUser

3. Fake - working implementation ที่ simplified
   - InMemoryUserRepository

4. Mock - verify interactions
   - verify(mockAnalytics).track(event)

5. Spy - wraps real implementation, records calls
   - spyk(realRepository)

เลือกใช้อะไร:
- Fake > Stub > Mock (เรียงลำดับความนิยม)
- Mock เมื่อต้องการ verify behavior
- Fake เมื่อ logic ซับซ้อน
- Stub เมื่อ simple return value
```

---

## ขั้นตอนที่ 1677: Fake Implementations

```kotlin
// ============================================
// Fake Repository สำหรับ Testing
// ============================================

class FakeProductRepository : ProductRepository {
    
    private val products = mutableMapOf<Long, Product>()
    private var shouldThrowError = false
    private var errorToThrow: Throwable = RuntimeException("Network error")
    
    // Test helpers
    fun addProduct(product: Product) { products[product.id] = product }
    fun setError(error: Throwable) { shouldThrowError = true; errorToThrow = error }
    fun clearError() { shouldThrowError = false }
    fun clear() { products.clear() }
    
    override fun observeProducts(): Flow<List<Product>> {
        return if (shouldThrowError) {
            flow { throw errorToThrow }
        } else {
            MutableStateFlow(products.values.toList())
        }
    }
    
    override suspend fun getProduct(id: Long): DomainResult<Product> {
        if (shouldThrowError) return DomainResult.Error(DomainError.NetworkError(errorToThrow.message ?: ""))
        
        return products[id]
            ?.let { DomainResult.Success(it) }
            ?: DomainResult.Error(DomainError.NotFound("Product", id))
    }
    
    override suspend fun searchProducts(query: String): DomainResult<List<Product>> {
        if (shouldThrowError) return DomainResult.Error(DomainError.NetworkError(""))
        
        val results = products.values.filter {
            it.name.contains(query, ignoreCase = true) ||
            it.description.contains(query, ignoreCase = true)
        }
        return DomainResult.Success(results)
    }
}

// ============================================
// Fake Time Provider
// ============================================

interface TimeProvider {
    fun now(): Instant
    fun today(): LocalDate
}

class FakeTimeProvider(
    private var currentTime: Instant = Instant.now()
) : TimeProvider {
    
    override fun now(): Instant = currentTime
    override fun today(): LocalDate = currentTime.atZone(ZoneId.systemDefault()).toLocalDate()
    
    fun advanceBy(duration: Duration) {
        currentTime = currentTime.plus(duration)
    }
    
    fun setTime(time: Instant) {
        currentTime = time
    }
}

// ============================================
// Test using Fakes
// ============================================

class OrderServiceTest {
    
    private val productRepository = FakeProductRepository()
    private val timeProvider = FakeTimeProvider()
    private val orderService = OrderService(productRepository, timeProvider)
    
    @Before
    fun setup() {
        productRepository.addProduct(
            Product(id = 1L, name = "iPhone", price = 35000.0, stock = 10)
        )
    }
    
    @Test
    fun `should create order successfully`() = runTest {
        val result = orderService.createOrder(
            productId = 1L,
            quantity = 2,
            userId = 100L
        )
        
        assertIs<DomainResult.Success<Order>>(result)
        assertEquals(70000.0, result.data.total)
    }
    
    @Test
    fun `should fail when product out of stock`() = runTest {
        productRepository.addProduct(
            Product(id = 2L, name = "iPad", price = 25000.0, stock = 0)
        )
        
        val result = orderService.createOrder(productId = 2L, quantity = 1, userId = 100L)
        
        assertIs<DomainResult.Error>(result)
        assertIs<DomainError.ValidationError>((result as DomainResult.Error).error)
    }
    
    @Test
    fun `should apply flash sale discount on weekends`() = runTest {
        // Set time to Saturday
        timeProvider.setTime(
            LocalDate.of(2024, 1, 6).atStartOfDay().toInstant(ZoneOffset.UTC)
        )
        
        val result = orderService.createOrder(productId = 1L, quantity = 1, userId = 100L)
        
        assertIs<DomainResult.Success<Order>>(result)
        assertTrue(result.data.total < 35000.0)  // discount applied
    }
}
```

---

## ขั้นตอนที่ 1678: Integration Testing with Room

```kotlin
// ============================================
// Room Integration Test
// ============================================

@RunWith(AndroidJUnit4::class)
class ProductDaoIntegrationTest {
    
    private lateinit var db: AppDatabase
    private lateinit var productDao: ProductDao
    
    @Before
    fun setup() {
        val context = ApplicationProvider.getApplicationContext<Context>()
        db = Room.inMemoryDatabaseBuilder(context, AppDatabase::class.java)
            .allowMainThreadQueries()
            .build()
        productDao = db.productDao()
    }
    
    @After
    fun teardown() {
        db.close()
    }
    
    @Test
    fun insertAndRetrieveProduct() = runTest {
        val product = ProductEntity(id = 1L, name = "Test Product", price = 100.0, categoryId = 1L)
        productDao.insert(product)
        
        val retrieved = productDao.getById(1L)
        
        assertNotNull(retrieved)
        assertEquals("Test Product", retrieved!!.name)
    }
    
    @Test
    fun searchProductsByName() = runTest {
        val products = listOf(
            ProductEntity(id = 1L, name = "iPhone 16", price = 35000.0, categoryId = 1L),
            ProductEntity(id = 2L, name = "iPad Pro", price = 28000.0, categoryId = 1L),
            ProductEntity(id = 3L, name = "MacBook Air", price = 45000.0, categoryId = 2L)
        )
        productDao.insertAll(products)
        
        val results = productDao.search("iPhone%")
        
        assertEquals(1, results.size)
        assertEquals("iPhone 16", results.first().name)
    }
    
    @Test
    fun observeProductsEmitsUpdates() = runTest {
        val products = mutableListOf<List<ProductEntity>>()
        
        val job = launch {
            productDao.observeAll().take(2).toList(products)
        }
        
        // Insert triggers emission
        productDao.insert(ProductEntity(id = 1L, name = "Product 1", price = 100.0, categoryId = 1L))
        
        job.join()
        
        assertEquals(2, products.size)  // empty + 1 item
        assertEquals(0, products[0].size)
        assertEquals(1, products[1].size)
    }
}
```

---

## ขั้นตอนที่ 1679: Screenshot Testing

```kotlin
// ============================================
// Compose Screenshot Tests (Paparazzi)
// ============================================

// implementation("app.cash.paparazzi:paparazzi:1.x")

@RunWith(JUnit4::class)
class ProductCardScreenshotTest {
    
    @get:Rule
    val paparazzi = Paparazzi(
        deviceConfig = DeviceConfig.PIXEL_6,
        theme = "android:Theme.Material3.Light.NoActionBar",
        showSystemUi = false
    )
    
    @Test
    fun productCard_default() {
        paparazzi.snapshot {
            AppTheme {
                ProductCard(
                    product = previewProduct,
                    onClick = {}
                )
            }
        }
    }
    
    @Test
    fun productCard_outOfStock() {
        paparazzi.snapshot {
            AppTheme {
                ProductCard(
                    product = previewProduct.copy(stock = 0),
                    onClick = {}
                )
            }
        }
    }
    
    @Test
    fun productCard_darkTheme() {
        paparazzi.snapshot {
            AppTheme(darkTheme = true) {
                ProductCard(
                    product = previewProduct,
                    onClick = {}
                )
            }
        }
    }
    
    @Test
    fun productCard_rtl() {
        paparazzi.snapshot(
            composable = {
                CompositionLocalProvider(LocalLayoutDirection provides LayoutDirection.Rtl) {
                    AppTheme {
                        ProductCard(product = previewProduct, onClick = {})
                    }
                }
            }
        )
    }
    
    private val previewProduct = Product(
        id = 1L,
        name = "iPhone 16 Pro Max",
        price = 49900.0,
        rating = 4.8f,
        reviewCount = 1234,
        imageUrl = "https://example.com/iphone.jpg",
        stock = 5
    )
}
```

---

## แบบฝึกหัด Part 88

```kotlin
// แบบฝึกหัด: TDD - Coupon System

// เขียน test ก่อน implementation ทุกครั้ง

// Test 1: Valid coupon reduces price
// Test 2: Expired coupon throws error
// Test 3: Max usage exceeded throws error
// Test 4: Minimum order amount not met throws error
// Test 5: Percentage discount calculated correctly
// Test 6: Fixed discount capped at total price

class CouponServiceTest {
    
    private val couponRepository = FakeCouponRepository()
    private val timeProvider = FakeTimeProvider()
    private val couponService = CouponService(couponRepository, timeProvider)
    
    @Test
    fun `valid coupon applies percentage discount`() = runTest {
        couponRepository.addCoupon(
            Coupon(
                code = "SAVE20",
                discountType = DiscountType.PERCENTAGE,
                discountValue = 20.0,
                minOrderAmount = 500.0,
                expiryDate = LocalDate.of(2099, 12, 31),
                maxUsageCount = 100,
                usageCount = 0
            )
        )
        
        val result = couponService.apply(code = "SAVE20", orderTotal = 1000.0)
        
        assertIs<CouponResult.Applied>(result)
        assertEquals(200.0, result.discountAmount)  // 20% of 1000
        assertEquals(800.0, result.finalTotal)
    }
    
    @Test
    fun `expired coupon returns error`() = runTest {
        timeProvider.setTime(LocalDate.of(2025, 1, 1).atStartOfDay().toInstant(ZoneOffset.UTC))
        couponRepository.addCoupon(
            Coupon(code = "OLD10", expiryDate = LocalDate.of(2024, 12, 31), /* ... */)
        )
        
        val result = couponService.apply(code = "OLD10", orderTotal = 1000.0)
        
        assertIs<CouponResult.Error>(result)
        assertEquals(CouponError.EXPIRED, result.error)
    }
    
    // TODO: implement all 6 test cases
}

sealed class CouponResult {
    data class Applied(val discountAmount: Double, val finalTotal: Double) : CouponResult()
    data class Error(val error: CouponError) : CouponResult()
}

enum class CouponError { EXPIRED, MAX_USAGE_EXCEEDED, MIN_ORDER_NOT_MET, NOT_FOUND }
```

---

*Part 88 จบแล้ว | ก่อนหน้า: [Part 87](../part87/README.md) | ถัดไป: [Part 89](../part89/README.md)*
