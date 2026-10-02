# Part 45: Android - DataStore Preferences
## ขั้นตอนที่ 901-925

---

## ขั้นตอนที่ 901: DataStore คืออะไร และต่างจาก SharedPreferences อย่างไร

DataStore คือ solution ใหม่แทน SharedPreferences ทำงานเป็น async (coroutine/Flow) ปลอดภัยกว่า

```kotlin
// build.gradle.kts
dependencies {
    // Preferences DataStore (Key-Value)
    implementation("androidx.datastore:datastore-preferences:1.0.0")

    // Proto DataStore (Typed - ใช้ Protocol Buffers)
    // implementation("androidx.datastore:datastore:1.0.0")
}

// ความแตกต่าง SharedPreferences vs DataStore
// SharedPreferences:
// - Synchronous (blocking main thread)
// - ไม่มี error handling
// - ไม่ thread-safe
// - เก็บได้แค่ primitive types

// DataStore:
// - Asynchronous (Flow/coroutine)
// - มี error handling
// - Thread-safe
// - Transaction-safe
```

---

## ขั้นตอนที่ 902: สร้าง DataStore

```kotlin
import android.content.Context
import androidx.datastore.core.DataStore
import androidx.datastore.preferences.core.*
import androidx.datastore.preferences.preferencesDataStore
import kotlinx.coroutines.flow.Flow
import kotlinx.coroutines.flow.catch
import kotlinx.coroutines.flow.map
import java.io.IOException

// สร้าง DataStore - เป็น extension property ของ Context
val Context.dataStore: DataStore<Preferences> by preferencesDataStore(name = "settings")

// กำหนด Keys
object PreferencesKeys {
    val THEME_MODE = stringPreferencesKey("theme_mode")
    val IS_NOTIFICATIONS_ENABLED = booleanPreferencesKey("notifications_enabled")
    val FONT_SIZE = floatPreferencesKey("font_size")
    val USER_ID = intPreferencesKey("user_id")
    val AUTH_TOKEN = stringPreferencesKey("auth_token")
    val LANGUAGE = stringPreferencesKey("language")
    val LAST_SYNC_TIME = longPreferencesKey("last_sync_time")
}

// Settings data class
data class AppSettings(
    val themeMode: String = "system",          // "light", "dark", "system"
    val isNotificationsEnabled: Boolean = true,
    val fontSize: Float = 1.0f,                // scale factor
    val language: String = "th"
)
```

---

## ขั้นตอนที่ 903: UserPreferencesRepository

```kotlin
class UserPreferencesRepository(private val context: Context) {

    private val dataStore = context.dataStore

    // อ่านค่าเป็น Flow - observe การเปลี่ยนแปลงตลอดเวลา
    val appSettings: Flow<AppSettings> = dataStore.data
        .catch { exception ->
            // จัดการ error เมื่ออ่าน datastore ไม่ได้
            if (exception is IOException) {
                emit(emptyPreferences())
            } else {
                throw exception
            }
        }
        .map { preferences ->
            AppSettings(
                themeMode = preferences[PreferencesKeys.THEME_MODE] ?: "system",
                isNotificationsEnabled = preferences[PreferencesKeys.IS_NOTIFICATIONS_ENABLED] ?: true,
                fontSize = preferences[PreferencesKeys.FONT_SIZE] ?: 1.0f,
                language = preferences[PreferencesKeys.LANGUAGE] ?: "th"
            )
        }

    // อ่านค่าเดี่ยว
    val authToken: Flow<String?> = dataStore.data
        .map { it[PreferencesKeys.AUTH_TOKEN] }

    val userId: Flow<Int?> = dataStore.data
        .map { it[PreferencesKeys.USER_ID] }

    // เขียนค่า
    suspend fun setThemeMode(mode: String) {
        dataStore.edit { preferences ->
            preferences[PreferencesKeys.THEME_MODE] = mode
        }
    }

    suspend fun setNotificationsEnabled(enabled: Boolean) {
        dataStore.edit { preferences ->
            preferences[PreferencesKeys.IS_NOTIFICATIONS_ENABLED] = enabled
        }
    }

    suspend fun setFontSize(size: Float) {
        dataStore.edit { preferences ->
            preferences[PreferencesKeys.FONT_SIZE] = size.coerceIn(0.8f, 1.4f)
        }
    }

    // บันทึก Auth Data
    suspend fun saveAuthData(userId: Int, token: String) {
        dataStore.edit { preferences ->
            preferences[PreferencesKeys.USER_ID] = userId
            preferences[PreferencesKeys.AUTH_TOKEN] = token
        }
    }

    // ล้าง Auth Data (logout)
    suspend fun clearAuthData() {
        dataStore.edit { preferences ->
            preferences.remove(PreferencesKeys.USER_ID)
            preferences.remove(PreferencesKeys.AUTH_TOKEN)
        }
    }

    // ล้างทุกอย่าง
    suspend fun clearAll() {
        dataStore.edit { it.clear() }
    }

    suspend fun updateLastSyncTime() {
        dataStore.edit { preferences ->
            preferences[PreferencesKeys.LAST_SYNC_TIME] = System.currentTimeMillis()
        }
    }
}
```

---

## ขั้นตอนที่ 904: ViewModel กับ DataStore

```kotlin
class SettingsViewModel(
    private val preferencesRepository: UserPreferencesRepository
) : ViewModel() {

    val settings: StateFlow<AppSettings> = preferencesRepository.appSettings
        .stateIn(
            scope = viewModelScope,
            started = SharingStarted.WhileSubscribed(5000),
            initialValue = AppSettings()
        )

    fun setTheme(mode: String) {
        viewModelScope.launch {
            preferencesRepository.setThemeMode(mode)
        }
    }

    fun toggleNotifications() {
        viewModelScope.launch {
            val current = settings.value.isNotificationsEnabled
            preferencesRepository.setNotificationsEnabled(!current)
        }
    }

    fun setFontSize(size: Float) {
        viewModelScope.launch {
            preferencesRepository.setFontSize(size)
        }
    }
}

@Composable
fun SettingsScreen(viewModel: SettingsViewModel = viewModel()) {
    val settings by viewModel.settings.collectAsStateWithLifecycle()

    LazyColumn(
        contentPadding = PaddingValues(16.dp),
        verticalArrangement = Arrangement.spacedBy(8.dp)
    ) {
        item {
            Text("การตั้งค่า", style = MaterialTheme.typography.headlineMedium)
        }

        item {
            // Theme selector
            Text("ธีม", style = MaterialTheme.typography.titleMedium)
            Row(horizontalArrangement = Arrangement.spacedBy(8.dp)) {
                listOf("light" to "สว่าง", "dark" to "มืด", "system" to "ระบบ")
                    .forEach { (value, label) ->
                        FilterChip(
                            selected = settings.themeMode == value,
                            onClick = { viewModel.setTheme(value) },
                            label = { Text(label) }
                        )
                    }
            }
        }

        item {
            // Notifications toggle
            Row(
                Modifier.fillMaxWidth(),
                Arrangement.SpaceBetween,
                Alignment.CenterVertically
            ) {
                Column {
                    Text("การแจ้งเตือน")
                    Text("รับการแจ้งเตือนจาก app", style = MaterialTheme.typography.bodySmall)
                }
                Switch(
                    checked = settings.isNotificationsEnabled,
                    onCheckedChange = { viewModel.toggleNotifications() }
                )
            }
        }

        item {
            // Font size
            Text("ขนาดตัวอักษร: ${(settings.fontSize * 100).toInt()}%")
            Slider(
                value = settings.fontSize,
                onValueChange = viewModel::setFontSize,
                valueRange = 0.8f..1.4f,
                steps = 5
            )
        }
    }
}
```

---

## ขั้นตอนที่ 905: DataStore กับ Hilt และ Proto DataStore

```kotlin
// Hilt Module สำหรับ DataStore
@Module
@InstallIn(SingletonComponent::class)
object DataStoreModule {
    @Provides
    @Singleton
    fun provideUserPreferencesRepository(
        @ApplicationContext context: Context
    ): UserPreferencesRepository = UserPreferencesRepository(context)
}

// ใช้ DataStore ที่ระดับ Application
class MyApplication : Application() {
    val userPreferencesRepository: UserPreferencesRepository by lazy {
        UserPreferencesRepository(this)
    }
}

// อ่านค่าใน MainActivity เพื่อตั้งค่า theme ก่อนแสดง UI
@AndroidEntryPoint
class MainActivity : ComponentActivity() {
    private val preferencesRepository by lazy {
        (applicationContext as MyApplication).userPreferencesRepository
    }

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)

        // อ่าน theme จาก DataStore และใช้ก่อนแสดง UI
        lifecycleScope.launch {
            preferencesRepository.appSettings.collect { settings ->
                setContent {
                    AppTheme(
                        darkTheme = when (settings.themeMode) {
                            "dark" -> true
                            "light" -> false
                            else -> isSystemInDarkTheme()
                        }
                    ) {
                        MainScreen()
                    }
                }
            }
        }
    }
}

// Encrypted DataStore - สำหรับข้อมูล sensitive
// ต้องเพิ่ม dependency:
// implementation("androidx.security:security-crypto-ktx:1.1.0-alpha06")

fun Context.encryptedDataStore(): DataStore<Preferences> {
    val masterKey = MasterKey.Builder(this)
        .setKeyScheme(MasterKey.KeyScheme.AES256_GCM)
        .build()

    return PreferenceDataStoreFactory.create(
        produceFile = {
            EncryptedFile.Builder(
                this,
                File(filesDir, "encrypted_settings.preferences_pb"),
                masterKey,
                EncryptedFile.FileEncryptionScheme.AES256_GCM_HKDF_4KB
            ).build().let { encFile ->
                encFile.openFileOutput() // เปิดใช้งาน
                File(filesDir, "encrypted_settings.preferences_pb")
            }
        }
    )
}
```

---

*Part 45 จบแล้ว | ก่อนหน้า: [Part 44](../part44/README.md) | ถัดไป: [Part 46](../part46/README.md)*
