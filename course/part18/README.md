# Part 18: File I/O และ DataStore
## ขั้นตอนที่ 421-450

---

## ขั้นตอนที่ 421: File Operations ใน Kotlin

```kotlin
import java.io.File
import java.nio.file.Files
import java.nio.file.Paths

// ============================================
// File Basics
// ============================================

fun fileBasics() {
    // สร้าง File object
    val file = File("data.txt")
    val absoluteFile = File("/home/user/data.txt")
    val relativeFile = File("subdir/data.txt")
    
    // ตรวจสอบ
    println(file.exists())          // false (ยังไม่ได้สร้าง)
    println(file.isFile)            // false
    println(file.isDirectory)       // false
    
    // สร้างไฟล์
    file.createNewFile()            // สร้างไฟล์เปล่า
    println(file.exists())          // true
    
    // Info
    println(file.name)              // data.txt
    println(file.extension)         // txt
    println(file.nameWithoutExtension) // data
    println(file.absolutePath)      // /current/dir/data.txt
    println(file.length())          // 0 bytes
    println(file.lastModified())    // timestamp
    
    // สร้าง directory
    val dir = File("mydir")
    dir.mkdir()                     // สร้าง 1 level
    
    val nestedDir = File("a/b/c")
    nestedDir.mkdirs()              // สร้างทุก level
    
    // ลบ
    file.delete()
    dir.deleteRecursively()         // ลบ directory และทุกอย่างใน
}

// ============================================
// เขียนไฟล์
// ============================================

fun writeFiles() {
    val file = File("output.txt")
    
    // writeText - เขียนทับ
    file.writeText("Hello World\n")
    file.writeText("Line 2\n", Charsets.UTF_8)
    
    // appendText - ต่อท้าย
    file.appendText("Line 3\n")
    
    // writeLines - เขียน list
    file.writeLines(listOf("item1", "item2", "item3"))
    
    // printWriter - มี println, printf
    file.printWriter().use { writer ->
        writer.println("Name: Alice")
        writer.println("Age: 25")
        writer.printf("Score: %.2f%n", 98.5)
    }  // use ปิด writer อัตโนมัติ
    
    // bufferedWriter - efficient สำหรับข้อมูลใหญ่
    file.bufferedWriter().use { writer ->
        repeat(10000) { i ->
            writer.write("Line $i\n")
        }
    }
    
    // Binary data
    val binaryFile = File("data.bin")
    binaryFile.writeBytes(byteArrayOf(0x48, 0x65, 0x6C, 0x6C, 0x6F))
    binaryFile.appendBytes(byteArrayOf(0x21))
}

// ============================================
// อ่านไฟล์
// ============================================

fun readFiles() {
    val file = File("data.txt")
    
    // readText - อ่านทั้งหมด
    val content = file.readText()
    val contentUtf8 = file.readText(Charsets.UTF_8)
    
    // readLines - อ่านทีละบรรทัด
    val lines = file.readLines()
    lines.forEach { println(it) }
    
    // forEachLine - efficient สำหรับไฟล์ใหญ่
    file.forEachLine { line ->
        println(line)
    }
    
    // bufferedReader
    file.bufferedReader().use { reader ->
        var line: String?
        while (reader.readLine().also { line = it } != null) {
            println(line)
        }
    }
    
    // Binary
    val bytes = file.readBytes()
    val hexString = bytes.joinToString(" ") { "%02X".format(it) }
    println(hexString)
    
    // useLines - lazy sequence (memory efficient)
    file.useLines { lines ->
        lines
            .filter { it.isNotBlank() }
            .map { it.trim() }
            .take(100)  // เอาแค่ 100 บรรทัด
            .forEach { println(it) }
    }
}
```

---

## ขั้นตอนที่ 422: CSV และ JSON Parsing

```kotlin
// ============================================
// CSV Parsing
// ============================================

data class CsvRecord(
    val id: Int,
    val name: String,
    val email: String,
    val score: Double
)

fun parseCsv(filePath: String): List<CsvRecord> {
    val file = File(filePath)
    if (!file.exists()) return emptyList()
    
    return file.readLines()
        .drop(1)  // skip header
        .filter { it.isNotBlank() }
        .mapNotNull { line ->
            val parts = line.split(",").map { it.trim() }
            if (parts.size >= 4) {
                try {
                    CsvRecord(
                        id = parts[0].toInt(),
                        name = parts[1],
                        email = parts[2],
                        score = parts[3].toDouble()
                    )
                } catch (e: NumberFormatException) {
                    null  // skip invalid rows
                }
            } else null
        }
}

fun writeCsv(records: List<CsvRecord>, filePath: String) {
    File(filePath).printWriter().use { writer ->
        writer.println("id,name,email,score")
        records.forEach { record ->
            writer.println("${record.id},${record.name},${record.email},${record.score}")
        }
    }
}

// ============================================
// JSON (ด้วย Gson)
// ============================================

data class Config(
    val apiUrl: String,
    val timeout: Int,
    val features: Map<String, Boolean>
)

fun readJsonConfig(filePath: String): Config? {
    val file = File(filePath)
    if (!file.exists()) return null
    
    return try {
        Gson().fromJson(file.readText(), Config::class.java)
    } catch (e: Exception) {
        println("Error parsing JSON: ${e.message}")
        null
    }
}

fun writeJsonConfig(config: Config, filePath: String) {
    val json = GsonBuilder()
        .setPrettyPrinting()
        .create()
        .toJson(config)
    
    File(filePath).writeText(json)
}
```

---

## ขั้นตอนที่ 423: Android Internal Storage

```kotlin
// Android มี storage หลายประเภท:
// 1. Internal Storage - private ต่อ app
// 2. External Storage - shared (ต้องขอ permission)
// 3. Cache - ลบได้เมื่อพื้นที่ไม่พอ
// 4. Assets - อ่านได้อย่างเดียว (packed ใน APK)

class FileManager(private val context: Context) {
    
    // ============================================
    // Internal Storage
    // ============================================
    
    fun writeToInternal(fileName: String, content: String) {
        // context.filesDir = /data/data/com.example/files/
        val file = File(context.filesDir, fileName)
        file.writeText(content)
    }
    
    fun readFromInternal(fileName: String): String? {
        val file = File(context.filesDir, fileName)
        return if (file.exists()) file.readText() else null
    }
    
    fun listInternalFiles(): List<String> {
        return context.filesDir.listFiles()?.map { it.name } ?: emptyList()
    }
    
    fun deleteInternalFile(fileName: String): Boolean {
        return File(context.filesDir, fileName).delete()
    }
    
    // ============================================
    // Cache Storage
    // ============================================
    
    fun writeToCashe(fileName: String, data: ByteArray) {
        // context.cacheDir = /data/data/com.example/cache/
        val file = File(context.cacheDir, fileName)
        file.writeBytes(data)
    }
    
    fun clearCache() {
        context.cacheDir.deleteRecursively()
        context.cacheDir.mkdirs()
    }
    
    fun getCacheSize(): Long {
        return context.cacheDir.walkBottomUp()
            .filter { it.isFile }
            .sumOf { it.length() }
    }
    
    // ============================================
    // Assets
    // ============================================
    
    fun readAsset(assetPath: String): String {
        return context.assets.open(assetPath).bufferedReader().readText()
    }
    
    fun readJsonAsset(assetPath: String): JsonObject {
        val content = readAsset(assetPath)
        return Gson().fromJson(content, JsonObject::class.java)
    }
    
    fun listAssets(directory: String = ""): List<String> {
        return context.assets.list(directory)?.toList() ?: emptyList()
    }
}
```

---

## ขั้นตอนที่ 424: DataStore Preferences

```kotlin
// DataStore ทดแทน SharedPreferences - safe กว่า, coroutine-based

// build.gradle.kts
// implementation("androidx.datastore:datastore-preferences:1.1.x")

// ============================================
// Preferences DataStore
// ============================================

// Keys
object UserPreferencesKeys {
    val THEME = booleanPreferencesKey("dark_theme")
    val LANGUAGE = stringPreferencesKey("language")
    val NOTIFICATION_ENABLED = booleanPreferencesKey("notifications")
    val FONT_SIZE = intPreferencesKey("font_size")
    val LAST_SYNC = longPreferencesKey("last_sync")
}

class UserPreferences(private val dataStore: DataStore<Preferences>) {
    
    // อ่านค่า
    val isDarkTheme: Flow<Boolean> = dataStore.data
        .catch { e ->
            if (e is IOException) emit(emptyPreferences())
            else throw e
        }
        .map { prefs ->
            prefs[UserPreferencesKeys.THEME] ?: false
        }
    
    val language: Flow<String> = dataStore.data
        .map { prefs -> prefs[UserPreferencesKeys.LANGUAGE] ?: "th" }
    
    val notificationsEnabled: Flow<Boolean> = dataStore.data
        .map { prefs -> prefs[UserPreferencesKeys.NOTIFICATION_ENABLED] ?: true }
    
    // อ่านทั้งหมด
    val userPrefs: Flow<UserPrefs> = dataStore.data
        .map { prefs ->
            UserPrefs(
                isDarkTheme = prefs[UserPreferencesKeys.THEME] ?: false,
                language = prefs[UserPreferencesKeys.LANGUAGE] ?: "th",
                notificationsEnabled = prefs[UserPreferencesKeys.NOTIFICATION_ENABLED] ?: true,
                fontSize = prefs[UserPreferencesKeys.FONT_SIZE] ?: 16
            )
        }
    
    // เขียนค่า
    suspend fun setDarkTheme(enabled: Boolean) {
        dataStore.edit { prefs ->
            prefs[UserPreferencesKeys.THEME] = enabled
        }
    }
    
    suspend fun setLanguage(language: String) {
        dataStore.edit { prefs ->
            prefs[UserPreferencesKeys.LANGUAGE] = language
        }
    }
    
    suspend fun setNotifications(enabled: Boolean) {
        dataStore.edit { prefs ->
            prefs[UserPreferencesKeys.NOTIFICATION_ENABLED] = enabled
        }
    }
    
    // Clear all
    suspend fun clearAll() {
        dataStore.edit { it.clear() }
    }
}

data class UserPrefs(
    val isDarkTheme: Boolean,
    val language: String,
    val notificationsEnabled: Boolean,
    val fontSize: Int
)

// Setup DataStore ด้วย Hilt
@Module
@InstallIn(SingletonComponent::class)
object DataStoreModule {
    
    @Provides
    @Singleton
    fun provideUserPreferences(@ApplicationContext context: Context): UserPreferences {
        val dataStore = PreferenceDataStoreFactory.create(
            corruptionHandler = ReplaceFileCorruptionHandler(
                produceNewData = { emptyPreferences() }
            ),
            scope = CoroutineScope(Dispatchers.IO + SupervisorJob()),
            produceFile = { context.preferencesDataStoreFile("user_preferences") }
        )
        return UserPreferences(dataStore)
    }
}
```

---

## ขั้นตอนที่ 425: Proto DataStore

```kotlin
// Proto DataStore - strongly typed ด้วย Protocol Buffers

// user_settings.proto (ใน src/main/proto/)
// syntax = "proto3";
// option java_package = "com.example.app";
// option java_multiple_files = true;
//
// message UserSettings {
//   bool dark_theme = 1;
//   string language = 2;
//   int32 font_size = 3;
//   bool notifications_enabled = 4;
// }

// Serializer
object UserSettingsSerializer : Serializer<UserSettings> {
    override val defaultValue: UserSettings = UserSettings.getDefaultInstance()
    
    override suspend fun readFrom(input: InputStream): UserSettings {
        try {
            return UserSettings.parseFrom(input)
        } catch (exception: InvalidProtocolBufferException) {
            throw CorruptionException("Cannot read proto.", exception)
        }
    }
    
    override suspend fun writeTo(t: UserSettings, output: OutputStream) {
        t.writeTo(output)
    }
}

// ใช้งาน Proto DataStore
class SettingsRepository(private val dataStore: DataStore<UserSettings>) {
    
    val settings: Flow<UserSettings> = dataStore.data
        .catch { e ->
            if (e is IOException) emit(UserSettings.getDefaultInstance())
            else throw e
        }
    
    suspend fun setDarkTheme(isDark: Boolean) {
        dataStore.updateData { current ->
            current.toBuilder().setDarkTheme(isDark).build()
        }
    }
    
    suspend fun setLanguage(language: String) {
        dataStore.updateData { current ->
            current.toBuilder().setLanguage(language).build()
        }
    }
}
```

---

## แบบฝึกหัด Part 18

```kotlin
// แบบฝึกหัด: Note App ที่บันทึกลง DataStore

data class Note(
    val id: String = UUID.randomUUID().toString(),
    val title: String,
    val content: String,
    val createdAt: Long = System.currentTimeMillis(),
    val updatedAt: Long = System.currentTimeMillis()
)

// TODO: สร้าง NoteDataStore ที่:
// - เก็บ List<Note> ใน DataStore (serialize เป็น JSON)
// - saveNote(note: Note)
// - deleteNote(id: String)
// - observeNotes(): Flow<List<Note>>
// - getNoteById(id: String): Note?
// - clearAll()

// Hint: ใช้ stringPreferencesKey และ Gson สำหรับ serialize
class NoteDataStore(private val dataStore: DataStore<Preferences>) {
    
    private val NOTES_KEY = stringPreferencesKey("notes")
    private val gson = Gson()
    
    fun observeNotes(): Flow<List<Note>> = dataStore.data
        .map { prefs ->
            val json = prefs[NOTES_KEY] ?: "[]"
            // TODO: deserialize from JSON
            emptyList()
        }
    
    suspend fun saveNote(note: Note) {
        dataStore.edit { prefs ->
            // TODO: get current notes, add/update, serialize
        }
    }
    
    // TODO: implement deleteNote, getNoteById, clearAll
}
```

---

*Part 18 จบแล้ว | ก่อนหน้า: [Part 17](../part17/README.md) | ถัดไป: [Part 19](../part19/README.md)*
