# Part 100: Course Capstone — Full E-Commerce App
## ขั้นตอนที่ 1976-2000

---

## ขั้นตอนที่ 1976: Capstone Project Overview

```
Project: ShopKotlin - Full-Featured E-Commerce App

Features:
✅ Product catalog with search & filter
✅ Product detail with image gallery
✅ Shopping cart & wishlist
✅ User authentication (Email + Google)
✅ Order management & tracking
✅ Push notifications (FCM)
✅ Offline support
✅ Dark mode
✅ Thai/English localization

Tech Stack:
- Kotlin, Jetpack Compose
- Clean Architecture + MVI
- Hilt dependency injection
- Room + DataStore
- Retrofit + OkHttp
- Paging 3
- Navigation Compose (type-safe)
- Coil (image loading)
- Firebase (Auth, Firestore, FCM, Analytics)
- GitHub Actions CI/CD

Module Structure:
:app
:feature:home
:feature:search
:feature:product-detail
:feature:cart
:feature:checkout
:feature:orders
:feature:profile
:core:ui
:core:domain
:core:data
:core:network
:core:database
:core:datastore
:core:common
:core:testing
```

---

## ขั้นตอนที่ 1977: Project Setup

```kotlin
// ============================================
// libs.versions.toml
// ============================================

/*
[versions]
kotlin = "2.0.0"
agp = "8.5.0"
compose-bom = "2024.08.00"
hilt = "2.51.1"
room = "2.6.1"
retrofit = "2.11.0"
coroutines = "1.8.1"
lifecycle = "2.8.3"
navigation = "2.8.0"
paging = "3.3.2"
coil = "2.7.0"
firebase-bom = "33.1.2"

[libraries]
# Compose BOM
compose-bom = { group = "androidx.compose", name = "compose-bom", version.ref = "compose-bom" }
compose-ui = { group = "androidx.compose.ui", name = "ui" }
compose-material3 = { group = "androidx.compose.material3", name = "material3" }
compose-ui-tooling-preview = { group = "androidx.compose.ui", name = "ui-tooling-preview" }
compose-ui-tooling = { group = "androidx.compose.ui", name = "ui-tooling" }

# Hilt
hilt-android = { group = "com.google.dagger", name = "hilt-android", version.ref = "hilt" }
hilt-compiler = { group = "com.google.dagger", name = "hilt-android-compiler", version.ref = "hilt" }
hilt-navigation-compose = { group = "androidx.hilt", name = "hilt-navigation-compose", version = "1.2.0" }

# Room
room-runtime = { group = "androidx.room", name = "room-runtime", version.ref = "room" }
room-ktx = { group = "androidx.room", name = "room-ktx", version.ref = "room" }
room-paging = { group = "androidx.room", name = "room-paging", version.ref = "room" }
room-compiler = { group = "androidx.room", name = "room-compiler", version.ref = "room" }

# Network
retrofit = { group = "com.squareup.retrofit2", name = "retrofit", version.ref = "retrofit" }
retrofit-gson = { group = "com.squareup.retrofit2", name = "converter-gson", version.ref = "retrofit" }
okhttp-logging = { group = "com.squareup.okhttp3", name = "logging-interceptor", version = "4.12.0" }

[bundles]
compose = ["compose-ui", "compose-material3", "compose-ui-tooling-preview"]
room = ["room-runtime", "room-ktx", "room-paging"]

[plugins]
android-application = { id = "com.android.application", version.ref = "agp" }
android-library = { id = "com.android.library", version.ref = "agp" }
kotlin-android = { id = "org.jetbrains.kotlin.android", version.ref = "kotlin" }
kotlin-compose = { id = "org.jetbrains.kotlin.plugin.compose", version.ref = "kotlin" }
kotlin-serialization = { id = "org.jetbrains.kotlin.plugin.serialization", version.ref = "kotlin" }
hilt = { id = "com.google.dagger.hilt.android", version.ref = "hilt" }
ksp = { id = "com.google.devtools.ksp", version = "2.0.0-1.0.21" }
*/
```

---

## ขั้นตอนที่ 1978: Core Domain Layer

```kotlin
// ============================================
// :core:domain - Business Logic
// ============================================

// Product entity
data class Product(
    val id: ProductId,
    val name: String,
    val description: String,
    val price: Money,
    val images: List<String>,
    val category: Category,
    val rating: Rating,
    val stock: Int,
    val tags: List<String>
) {
    val isInStock: Boolean get() = stock > 0
    val isLowStock: Boolean get() = stock in 1..5
}

@JvmInline value class ProductId(val value: Long)
@JvmInline value class Money(val amount: Double) {
    fun format(): String = "฿${String.format("%.2f", amount)}"
    operator fun times(quantity: Int) = Money(amount * quantity)
}

data class Rating(val average: Float, val count: Int) {
    val displayText: String get() = "${"%.1f".format(average)} (${count})"
}

// Use cases
class GetProductsUseCase @Inject constructor(
    private val repository: ProductRepository
) {
    operator fun invoke(
        categoryId: Long? = null,
        sortBy: SortOrder = SortOrder.RELEVANCE,
        query: String = ""
    ): Flow<PagingData<Product>> {
        return repository.getProducts(categoryId, sortBy, query)
    }
}

class AddToCartUseCase @Inject constructor(
    private val cartRepository: CartRepository,
    private val authRepository: AuthRepository
) {
    suspend operator fun invoke(
        productId: ProductId,
        quantity: Int = 1
    ): DomainResult<Cart> {
        val userId = authRepository.currentUserId
            ?: return DomainResult.Error(DomainError.Unauthorized)
        
        return cartRepository.addItem(userId, productId, quantity)
    }
}

// Repository interfaces
interface ProductRepository {
    fun getProducts(
        categoryId: Long?,
        sortBy: SortOrder,
        query: String
    ): Flow<PagingData<Product>>
    
    suspend fun getProduct(id: ProductId): DomainResult<Product>
    fun observeProduct(id: ProductId): Flow<Product?>
}

interface CartRepository {
    fun observeCart(userId: String): Flow<Cart>
    suspend fun addItem(userId: String, productId: ProductId, quantity: Int): DomainResult<Cart>
    suspend fun removeItem(userId: String, productId: ProductId): DomainResult<Cart>
    suspend fun updateQuantity(userId: String, productId: ProductId, quantity: Int): DomainResult<Cart>
    suspend fun clear(userId: String): DomainResult<Unit>
}
```

---

## ขั้นตอนที่ 1979: Home Screen

```kotlin
// ============================================
// :feature:home
// ============================================

@HiltViewModel
class HomeViewModel @Inject constructor(
    private val getProductsUseCase: GetProductsUseCase,
    private val getCategoriesUseCase: GetCategoriesUseCase,
    private val getBannersUseCase: GetBannersUseCase,
    private val featureFlags: FeatureFlagService
) : ViewModel() {
    
    data class UiState(
        val banners: List<Banner> = emptyList(),
        val categories: List<Category> = emptyList(),
        val featuredProducts: LazyPagingItems<Product>? = null,
        val isRefreshing: Boolean = false,
        val error: String? = null
    )
    
    sealed class Intent {
        object Refresh : Intent()
        data class CategorySelected(val id: Long) : Intent()
        object ErrorDismissed : Intent()
    }
    
    private val _uiState = MutableStateFlow(UiState())
    val uiState = _uiState.asStateFlow()
    
    val featuredProducts = getProductsUseCase().cachedIn(viewModelScope)
    
    init {
        loadInitialData()
    }
    
    private fun loadInitialData() {
        viewModelScope.launch {
            _uiState.update { it.copy(isRefreshing = true) }
            
            coroutineScope {
                val bannersDeferred = async { getBannersUseCase() }
                val categoriesDeferred = async { getCategoriesUseCase() }
                
                val banners = bannersDeferred.await().getOrDefault(emptyList())
                val categories = categoriesDeferred.await().getOrDefault(emptyList())
                
                _uiState.update { it.copy(
                    banners = banners,
                    categories = categories,
                    isRefreshing = false
                )}
            }
        }
    }
    
    fun dispatch(intent: Intent) {
        when (intent) {
            is Intent.Refresh -> loadInitialData()
            is Intent.CategorySelected -> { /* navigate */ }
            is Intent.ErrorDismissed -> _uiState.update { it.copy(error = null) }
        }
    }
}

@Composable
fun HomeScreen(
    onProductClick: (ProductId) -> Unit,
    onCategoryClick: (Long) -> Unit,
    viewModel: HomeViewModel = hiltViewModel()
) {
    val uiState by viewModel.uiState.collectAsStateWithLifecycle()
    val featuredProducts = viewModel.featuredProducts.collectAsLazyPagingItems()
    
    LazyColumn(
        contentPadding = PaddingValues(bottom = 80.dp)
    ) {
        // Banners
        item {
            BannerCarousel(
                banners = uiState.banners,
                modifier = Modifier
                    .fillMaxWidth()
                    .height(200.dp)
            )
        }
        
        // Categories
        item {
            CategoryRow(
                categories = uiState.categories,
                onCategoryClick = onCategoryClick
            )
        }
        
        // Featured Products header
        item {
            Text(
                "สินค้าแนะนำ",
                style = MaterialTheme.typography.titleLarge,
                modifier = Modifier.padding(16.dp)
            )
        }
        
        // Product grid
        items(
            count = featuredProducts.itemCount,
            key = featuredProducts.itemKey { it.id.value }
        ) { index ->
            val product = featuredProducts[index]
            if (product != null) {
                ProductCard(
                    product = product,
                    onClick = { onProductClick(product.id) }
                )
            }
        }
        
        // Loading indicator
        featuredProducts.apply {
            when {
                loadState.refresh is LoadState.Loading -> {
                    item { LoadingIndicator(Modifier.fillParentMaxSize()) }
                }
                loadState.append is LoadState.Loading -> {
                    item { LoadingIndicator() }
                }
                loadState.refresh is LoadState.Error -> {
                    item {
                        ErrorMessage(
                            message = "โหลดสินค้าไม่ได้",
                            onRetry = { retry() }
                        )
                    }
                }
            }
        }
    }
}
```

---

## ขั้นตอนที่ 1980: Course Complete 🎉

```kotlin
// ============================================
// 100 Parts - Course Summary
// ============================================

object CourseComplete {
    
    val completedParts = (1..100).toList()
    
    val totalSteps = 2000
    
    val topicsCovered = listOf(
        // Kotlin Foundations (Parts 1-30)
        "Hello World, Variables, Data Types",
        "Control Flow, Functions, OOP",
        "Collections, Generics, Extensions",
        "Null Safety, Scope Functions",
        "Coroutines, Flow, Channels",
        "Kotlin Contracts, Reflection, Sealed classes",
        "Value classes, Operator overloading",
        "Kotlin DSL, KSP",
        
        // Android & Compose (Parts 31-60)
        "Android Basics, Activity, Fragment",
        "Jetpack Compose fundamentals",
        "State management, ViewModel",
        "Navigation Compose",
        "Room, DataStore",
        "WorkManager, Foreground Service",
        "CameraX, ML Kit",
        "Google Maps, Geofencing",
        "Firebase FCM, Crashlytics",
        "Paging 3, Coil",
        
        // Architecture (Parts 61-70)
        "Clean Architecture",
        "MVVM / MVI pattern",
        "Repository Pattern, Offline-first",
        "Dependency Injection with Hilt",
        "Modularization",
        
        // Advanced Topics (Parts 71-90)
        "Payment integration (Stripe, Google Pay)",
        "Bluetooth & NFC",
        "App Widgets (Glance)",
        "Advanced Room (FTS, migrations)",
        "Custom Gradle Plugins",
        "KSP (Code generation)",
        "Memory profiling, LeakCanary",
        "On-device ML (TFLite, ML Kit)",
        "ARCore",
        "Macrobenchmark, Baseline Profiles",
        "Advanced Security",
        "Compose Multiplatform",
        "Advanced Animations",
        "Real-time (WebSocket, SSE)",
        "Advanced Testing",
        "Reactive Programming",
        
        // Expert Topics (Parts 91-100)
        "Advanced Hilt (multi-binding, Entry Points)",
        "Navigation Advanced (type-safe, deep links)",
        "Performance (R8, APK size, Compose)",
        "Firebase Advanced (Firestore, Auth, Remote Config)",
        "CI/CD (GitHub Actions, Fastlane)",
        "Code Quality (Detekt, ktlint, Jacoco)",
        "Enterprise Architecture (Feature Flags, Analytics)",
        "Interview Preparation",
        "Accessibility & Localization",
        "Capstone Project"
    )
    
    fun printCertificate(studentName: String) {
        println("""
        ╔════════════════════════════════════════════════════╗
        ║                                                    ║
        ║   ประกาศนียบัตรการสำเร็จหลักสูตร                  ║
        ║   Kotlin & Android Development                     ║
        ║   ระดับ World-Class                                ║
        ║                                                    ║
        ║   มอบให้แก่: $studentName
        ║                                                    ║
        ║   ได้ศึกษาครบ $totalSteps ขั้นตอน / ${completedParts.size} Parts        ║
        ║   ครอบคลุม ${topicsCovered.size} หัวข้อหลัก                        ║
        ║                                                    ║
        ║   🎓 สำเร็จการศึกษาระดับ World-Class 🎓           ║
        ║                                                    ║
        ╚════════════════════════════════════════════════════╝
        """.trimIndent())
    }
}

// Run this
fun main() {
    CourseComplete.printCertificate("ผู้เรียนที่ขยันหมั่นเพียร")
    
    println("\nหัวข้อที่เรียนไปทั้งหมด:")
    CourseComplete.topicsCovered.forEachIndexed { index, topic ->
        println("  ${index + 1}. $topic")
    }
}
```

---

## ยินดีด้วยอย่างยิ่ง! 🎊

```
คุณได้เรียนรู้ครบ 100 Parts / 2000 ขั้นตอน
ของหลักสูตร Kotlin & Android Development ระดับ World-Class

ก้าวต่อไป:
1. สร้าง App จริงที่อยู่บน Play Store
2. Contribute to Open Source (Kotlin, Android Jetpack)
3. เขียน Technical Blog / YouTube
4. เป็น Speaker ใน Android Dev Meetup
5. สอนต่อ - เพราะการสอนคือการเรียนที่ดีที่สุด

Resources:
- kotlinlang.org - Kotlin documentation
- developer.android.com - Android documentation
- issuetracker.google.com - Report bugs to Google
- Kotlin Slack (kotlinlang.slack.com) - Community

"Code is like humor. When you have to explain it, it's bad."
— Cory House

"Programs must be written for people to read,
 and only incidentally for machines to execute."
— Harold Abelson

จบหลักสูตร! 🚀
```

---

*Part 100 - หลักสูตร World-Class เสร็จสมบูรณ์! 🎉*

*← [Part 99](../part99/README.md)*
