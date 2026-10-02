# Part 87: Modularization Strategy
## ขั้นตอนที่ 1651-1675

---

## ขั้นตอนที่ 1651: Why Modularize

```
Benefits ของ Modularization:

1. Faster builds
   - Only rebuild changed modules
   - Parallel compilation
   - Better cache hits

2. Scalability
   - Multiple teams work independently
   - Clear ownership

3. Reusability
   - Share code across apps
   - KMP: share with iOS

4. Testability
   - Test modules in isolation
   - Easier mocking

Module Types:
:app         - entry point, ties everything together
:feature:X   - vertical slice (screen + logic)
:core:X      - horizontal slice (shared utilities)
:library:X   - external library wrappers
```

---

## ขั้นตอนที่ 1652: Module Graph Design

```
Recommended Module Graph (Now in Android style):

:app
├── :feature:home
├── :feature:search
├── :feature:profile
├── :feature:cart
├── :feature:orders
└── :feature:settings

:feature:* depends on:
├── :core:ui         (Compose components, design system)
├── :core:domain     (Use cases, models)
├── :core:data       (Repositories)
└── :core:common     (Utils, extensions)

:core:data depends on:
├── :core:network    (Retrofit, OkHttp)
├── :core:database   (Room)
├── :core:datastore  (DataStore)
└── :core:model      (Data classes)

:core:domain depends on:
└── :core:model only

:core:network depends on:
└── :core:common

Rule: No circular dependencies!
```

---

## ขั้นตอนที่ 1653: Feature Module Structure

```
:feature:product module structure:

feature/product/
├── src/
│   ├── main/
│   │   ├── kotlin/
│   │   │   └── com/myapp/feature/product/
│   │   │       ├── di/
│   │   │       │   └── ProductModule.kt
│   │   │       ├── navigation/
│   │   │       │   └── ProductNavigation.kt
│   │   │       ├── ui/
│   │   │       │   ├── list/
│   │   │       │   │   ├── ProductListScreen.kt
│   │   │       │   │   └── ProductListViewModel.kt
│   │   │       │   └── detail/
│   │   │       │       ├── ProductDetailScreen.kt
│   │   │       │       └── ProductDetailViewModel.kt
│   │   │       └── ProductFeature.kt  (public API)
│   └── test/
│       └── ProductListViewModelTest.kt
└── build.gradle.kts
```

```kotlin
// ProductNavigation.kt - public API ของ feature
const val PRODUCT_LIST_ROUTE = "product_list"
const val PRODUCT_DETAIL_ROUTE = "product_detail/{productId}"

fun NavController.navigateToProductList() {
    navigate(PRODUCT_LIST_ROUTE)
}

fun NavController.navigateToProductDetail(productId: Long) {
    navigate("product_detail/$productId")
}

fun NavGraphBuilder.productGraph(
    navController: NavHostController
) {
    composable(PRODUCT_LIST_ROUTE) {
        ProductListScreen(
            onProductClick = { id -> navController.navigateToProductDetail(id) },
            onCartClick = { navController.navigate("cart") }
        )
    }
    
    composable(
        route = PRODUCT_DETAIL_ROUTE,
        arguments = listOf(navArgument("productId") { type = NavType.LongType })
    ) { backStackEntry ->
        val productId = backStackEntry.arguments?.getLong("productId") ?: return@composable
        
        ProductDetailScreen(
            productId = productId,
            onBack = { navController.popBackStack() },
            onAddToCart = { navController.navigate("cart") }
        )
    }
}
```

---

## ขั้นตอนที่ 1654: Dependency Injection Across Modules

```kotlin
// ============================================
// Hilt across modules
// ============================================

// :core:data module
@Module
@InstallIn(SingletonComponent::class)
object DataModule {
    
    @Provides
    @Singleton
    fun provideProductRepository(
        api: ProductApi,
        dao: ProductDao,
        networkMonitor: NetworkMonitor
    ): ProductRepository {
        return ProductRepositoryImpl(api, dao, networkMonitor)
    }
}

// :core:domain module
@Module
@InstallIn(ViewModelComponent::class)
object DomainModule {
    
    @Provides
    @ViewModelScoped
    fun provideGetProductsUseCase(
        repository: ProductRepository
    ): GetProductsUseCase {
        return GetProductsUseCase(repository)
    }
}

// :feature:product module
@HiltViewModel
class ProductListViewModel @Inject constructor(
    private val getProductsUseCase: GetProductsUseCase  // from :core:domain
) : ViewModel() {
    // ...
}

// :app module - register all components
@HiltAndroidApp
class MyApplication : Application()
```

---

## ขั้นตอนที่ 1655: Module Boundaries & API Surface

```kotlin
// ============================================
// Controlling public API
// ============================================

// ทุก public class/function ใน feature module ควร limit
// ใช้ internal keyword สำหรับ implementation details

// feature/product/ProductNavigation.kt - PUBLIC
const val PRODUCT_LIST_ROUTE = "product_list"
fun NavController.navigateToProductList() { /* ... */ }

// feature/product/ui/list/ProductListScreen.kt - PUBLIC
@Composable
fun ProductListScreen(
    onProductClick: (Long) -> Unit,
    onCartClick: () -> Unit
) { /* ... */ }

// feature/product/ui/list/ProductListViewModel.kt - INTERNAL
internal class ProductListViewModel : ViewModel() { /* ... */ }

// feature/product/di/ProductModule.kt - INTERNAL (สำหรับ Hilt only)
@Module
@InstallIn(ViewModelComponent::class)
internal object ProductModule { /* ... */ }

// ============================================
// Explicit module API (optional)
// ============================================

// ประกาศ public API ของ module ไว้ใน 1 file
object ProductFeature {
    
    // Navigation
    const val ROUTE = PRODUCT_LIST_ROUTE
    fun NavController.navigate() = navigateToProductList()
    
    // Composable screens
    val listScreen: @Composable (onProductClick: (Long) -> Unit) -> Unit = { onClick ->
        ProductListScreen(onProductClick = onClick, onCartClick = {})
    }
    
    // NavGraph builder
    fun NavGraphBuilder.register(navController: NavHostController) {
        productGraph(navController)
    }
}
```

---

## แบบฝึกหัด Part 87

```kotlin
// แบบฝึกหัด: สร้าง :feature:wishlist module

// Module structure:
// :feature:wishlist
// - WishlistScreen (แสดงรายการ wishlist)
// - WishlistViewModel
// - WishlistNavigation (route, NavGraphBuilder extension)
// - ToggleWishlistUseCase (เพิ่ม/ลบ จาก :core:domain)

// ใน :core:domain
interface WishlistRepository {
    fun observeWishlist(userId: Long): Flow<List<Product>>
    suspend fun toggleWishlist(userId: Long, productId: Long): Boolean  // returns new state
    suspend fun isWishlisted(userId: Long, productId: Long): Boolean
}

// ใน :feature:wishlist
const val WISHLIST_ROUTE = "wishlist"

fun NavController.navigateToWishlist() = navigate(WISHLIST_ROUTE)

fun NavGraphBuilder.wishlistGraph(navController: NavHostController) {
    composable(WISHLIST_ROUTE) {
        WishlistScreen(
            onProductClick = { id -> navController.navigateToProductDetail(id) },
            onBack = { navController.popBackStack() }
        )
    }
}

// ViewModel
@HiltViewModel
class WishlistViewModel @Inject constructor(
    private val wishlistRepository: WishlistRepository,
    private val userPreferences: UserPreferences
) : ViewModel() {
    // TODO: implement
}
```

---

*Part 87 จบแล้ว | ก่อนหน้า: [Part 86](../part86/README.md) | ถัดไป: [Part 88](../part88/README.md)*
