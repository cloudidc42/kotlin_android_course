# Part 53: Modularization
## ขั้นตอนที่ 801-825

---

## ขั้นตอนที่ 801: Modularization คืออะไร?

แบ่ง app เป็น modules อิสระเพื่อ build เร็วขึ้น, reuse code ได้, และ maintain ง่าย

```
Modular Architecture:

app/
├── :app                    (entry point)
│
├── feature/
│   ├── :feature:home       (Home screen)
│   ├── :feature:profile    (Profile screen)
│   ├── :feature:search     (Search screen)
│   └── :feature:settings   (Settings screen)
│
├── core/
│   ├── :core:ui            (Shared UI components)
│   ├── :core:network       (Retrofit, OkHttp)
│   ├── :core:database      (Room)
│   ├── :core:common        (Extensions, utilities)
│   └── :core:testing       (Test utilities)
│
└── domain/
    ├── :domain:user        (User domain logic)
    ├── :domain:product     (Product domain logic)
    └── :domain:order       (Order domain logic)

Dependency:
:app → :feature:* → :domain:* → :core:*
```

---

## ขั้นตอนที่ 802: Settings ของ Multi-module Project

```kotlin
// settings.gradle.kts
pluginManagement {
    repositories {
        google()
        mavenCentral()
        gradlePluginPortal()
    }
}

dependencyResolutionManagement {
    repositoriesMode.set(RepositoriesMode.FAIL_ON_PROJECT_REPOS)
    repositories {
        google()
        mavenCentral()
    }
}

rootProject.name = "MyApp"

// ประกาศ modules
include(":app")
include(":feature:home")
include(":feature:profile")
include(":feature:search")
include(":core:ui")
include(":core:network")
include(":core:database")
include(":core:common")
include(":domain:user")
include(":domain:product")
```

---

## ขั้นตอนที่ 803: Convention Plugins

```kotlin
// build-logic/convention/src/main/kotlin/

// AndroidLibraryConventionPlugin.kt
class AndroidLibraryConventionPlugin : Plugin<Project> {
    override fun apply(target: Project) {
        with(target) {
            with(pluginManager) {
                apply("com.android.library")
                apply("org.jetbrains.kotlin.android")
            }
            
            extensions.configure<LibraryExtension> {
                compileSdk = 35
                defaultConfig {
                    minSdk = 26
                    testInstrumentationRunner = "androidx.test.runner.AndroidJUnitRunner"
                }
                
                compileOptions {
                    sourceCompatibility = JavaVersion.VERSION_17
                    targetCompatibility = JavaVersion.VERSION_17
                }
            }
        }
    }
}

// AndroidFeatureConventionPlugin.kt
class AndroidFeatureConventionPlugin : Plugin<Project> {
    override fun apply(target: Project) {
        with(target) {
            pluginManager.apply {
                apply("myapp.android.library")
                apply("myapp.android.hilt")
            }
            
            dependencies {
                add("implementation", project(":core:ui"))
                add("implementation", project(":core:common"))
                add("testImplementation", project(":core:testing"))
            }
        }
    }
}

// HiltConventionPlugin.kt
class HiltConventionPlugin : Plugin<Project> {
    override fun apply(target: Project) {
        with(target) {
            pluginManager.apply("com.google.dagger.hilt.android")
            dependencies {
                add("implementation", libs.findLibrary("hilt.android").get())
                add("ksp", libs.findLibrary("hilt.compiler").get())
            }
        }
    }
}

// build-logic/convention/build.gradle.kts
gradlePlugin {
    plugins {
        register("androidLibrary") {
            id = "myapp.android.library"
            implementationClass = "AndroidLibraryConventionPlugin"
        }
        register("androidFeature") {
            id = "myapp.android.feature"
            implementationClass = "AndroidFeatureConventionPlugin"
        }
        register("hilt") {
            id = "myapp.android.hilt"
            implementationClass = "HiltConventionPlugin"
        }
    }
}
```

---

## ขั้นตอนที่ 804: Feature Module Structure

```kotlin
// feature/home/build.gradle.kts
plugins {
    alias(libs.plugins.myapp.android.feature)
    alias(libs.plugins.myapp.android.library.compose)
}

dependencies {
    implementation(project(":domain:product"))
    implementation(project(":domain:user"))
}

// feature/home/src/main/java/com/example/feature/home/

// HomeScreen.kt
@Composable
fun HomeRoute(
    navController: NavController,
    viewModel: HomeViewModel = hiltViewModel()
) {
    val state by viewModel.state.collectAsStateWithLifecycle()
    
    HomeScreen(
        state = state,
        onProductClick = { product ->
            navController.navigate("product/${product.id}")
        },
        onSeeAllProducts = {
            navController.navigate("products")
        }
    )
}

@Composable
internal fun HomeScreen(
    state: HomeState,
    onProductClick: (Product) -> Unit,
    onSeeAllProducts: () -> Unit
) {
    // UI implementation
}

// HomeNavigation.kt
fun NavGraphBuilder.homeScreen(navController: NavController) {
    composable("home") {
        HomeRoute(navController = navController)
    }
}

fun NavController.navigateToHome() {
    navigate("home")
}
```

---

## ขั้นตอนที่ 805: Core Module Structure

```kotlin
// core/ui/build.gradle.kts
plugins {
    alias(libs.plugins.myapp.android.library)
    alias(libs.plugins.myapp.android.library.compose)
}

// Shared UI components
// core/ui/src/main/java/com/example/core/ui/

// components/LoadingView.kt
@Composable
fun LoadingView(
    modifier: Modifier = Modifier,
    text: String = "กำลังโหลด..."
) {
    Box(modifier = modifier.fillMaxSize(), contentAlignment = Alignment.Center) {
        Column(horizontalAlignment = Alignment.CenterHorizontally) {
            CircularProgressIndicator()
            Spacer(Modifier.height(8.dp))
            Text(text, style = MaterialTheme.typography.bodyMedium)
        }
    }
}

// components/ErrorView.kt
@Composable
fun ErrorView(
    error: String,
    onRetry: (() -> Unit)? = null,
    modifier: Modifier = Modifier
) {
    Column(
        modifier = modifier.padding(16.dp),
        horizontalAlignment = Alignment.CenterHorizontally
    ) {
        Icon(
            Icons.Default.Warning,
            contentDescription = null,
            tint = MaterialTheme.colorScheme.error,
            modifier = Modifier.size(48.dp)
        )
        Spacer(Modifier.height(8.dp))
        Text(
            error,
            style = MaterialTheme.typography.bodyMedium,
            color = MaterialTheme.colorScheme.error,
            textAlign = TextAlign.Center
        )
        onRetry?.let {
            Spacer(Modifier.height(16.dp))
            Button(onClick = it) { Text("ลองใหม่") }
        }
    }
}

// components/EmptyView.kt
@Composable
fun EmptyView(
    title: String = "ไม่มีข้อมูล",
    message: String = "",
    icon: ImageVector = Icons.Default.Inbox,
    action: (@Composable () -> Unit)? = null,
    modifier: Modifier = Modifier
) {
    Column(
        modifier = modifier.padding(24.dp),
        horizontalAlignment = Alignment.CenterHorizontally
    ) {
        Icon(
            icon,
            contentDescription = null,
            modifier = Modifier.size(64.dp),
            tint = MaterialTheme.colorScheme.onSurfaceVariant
        )
        Spacer(Modifier.height(16.dp))
        Text(title, style = MaterialTheme.typography.titleMedium)
        if (message.isNotBlank()) {
            Spacer(Modifier.height(8.dp))
            Text(
                message,
                style = MaterialTheme.typography.bodyMedium,
                color = MaterialTheme.colorScheme.onSurfaceVariant,
                textAlign = TextAlign.Center
            )
        }
        action?.let {
            Spacer(Modifier.height(16.dp))
            it()
        }
    }
}

// theme/Theme.kt
@Composable
fun AppTheme(
    darkTheme: Boolean = isSystemInDarkTheme(),
    dynamicColor: Boolean = true,
    content: @Composable () -> Unit
) {
    val colorScheme = when {
        dynamicColor && Build.VERSION.SDK_INT >= Build.VERSION_CODES.S -> {
            val context = LocalContext.current
            if (darkTheme) dynamicDarkColorScheme(context)
            else dynamicLightColorScheme(context)
        }
        darkTheme -> DarkColorScheme
        else -> LightColorScheme
    }
    
    MaterialTheme(
        colorScheme = colorScheme,
        typography = AppTypography,
        content = content
    )
}
```

---

## ขั้นตอนที่ 806: App Module Navigation

```kotlin
// app/src/main/java/com/example/app/

// AppNavigation.kt
@Composable
fun AppNavigation() {
    val navController = rememberNavController()
    
    NavHost(navController = navController, startDestination = "home") {
        // Feature modules register their own nav graphs
        homeScreen(navController)
        profileScreen(navController)
        searchScreen(navController)
        settingsScreen(navController)
        
        // Deep links
        composable(
            route = "product/{id}",
            deepLinks = listOf(navDeepLink {
                uriPattern = "https://example.com/product/{id}"
            })
        ) { entry ->
            val id = entry.arguments!!.getString("id")!!.toLong()
            ProductDetailRoute(productId = id, navController = navController)
        }
    }
}

// AppViewModel.kt - App-level state
@HiltViewModel
class AppViewModel @Inject constructor(
    private val authRepository: AuthRepository,
    private val userPreferences: UserPreferences
) : ViewModel() {
    
    val isLoggedIn: StateFlow<Boolean> = authRepository
        .observeAuthState()
        .stateIn(viewModelScope, SharingStarted.Eagerly, false)
    
    val isDarkTheme: StateFlow<Boolean> = userPreferences
        .observeDarkTheme()
        .stateIn(viewModelScope, SharingStarted.Eagerly, false)
}

// MainActivity.kt
@AndroidEntryPoint
class MainActivity : ComponentActivity() {
    
    private val viewModel: AppViewModel by viewModels()
    
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        enableEdgeToEdge()
        
        setContent {
            val isDarkTheme by viewModel.isDarkTheme.collectAsStateWithLifecycle()
            val isLoggedIn by viewModel.isLoggedIn.collectAsStateWithLifecycle()
            
            AppTheme(darkTheme = isDarkTheme) {
                if (isLoggedIn) {
                    AppNavigation()
                } else {
                    AuthNavigation()
                }
            }
        }
    }
}
```

---

## แบบฝึกหัด Part 53

```kotlin
// แบบฝึกหัด: วางแผน Modular Architecture สำหรับ E-commerce App

// App: Online Shopping App
// Features: Home, Product List, Product Detail, Cart, Checkout, Order History, Profile, Settings

// TODO: วาด dependency graph ของ modules
// TODO: กำหนด Convention Plugins ที่ต้องการ
// TODO: สร้าง module structure:
//   - :app
//   - :feature:home
//   - :feature:shop (product list + detail)
//   - :feature:cart
//   - :feature:checkout
//   - :feature:orders
//   - :feature:profile
//   - :core:ui
//   - :core:network
//   - :core:database
//   - :core:common
//   - :domain:product
//   - :domain:cart
//   - :domain:order
//   - :domain:user

// สร้าง :core:common module ที่มี:
// - Extension functions ที่ใช้บ่อย
// - Base classes (BaseViewModel, UseCase)
// - Result, Resource sealed classes
// - Date/Time utilities
// - Currency formatting
```

---

*Part 53 จบแล้ว | ก่อนหน้า: [Part 52](../part52/README.md) | ถัดไป: [Part 54](../part54/README.md)*
