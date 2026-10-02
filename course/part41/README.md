# Part 41: Android - Room Database พื้นฐาน
## ขั้นตอนที่ 801-825

---

## ขั้นตอนที่ 801: ติดตั้งและตั้งค่า Room

Room เป็น ORM (Object-Relational Mapping) บน SQLite ของ Android ช่วยทำงานกับ database ได้ง่าย

```kotlin
// build.gradle.kts (app)
plugins {
    id("com.google.devtools.ksp") version "1.9.0-1.0.13"
}

dependencies {
    val roomVersion = "2.6.1"
    implementation("androidx.room:room-runtime:$roomVersion")
    implementation("androidx.room:room-ktx:$roomVersion")
    ksp("androidx.room:room-compiler:$roomVersion")
}

// Room มีส่วนประกอบหลัก 3 อย่าง:
// 1. @Entity   - แทน Table ใน database
// 2. @Dao      - Data Access Object - คำสั่ง SQL
// 3. @Database - database หลัก เชื่อม Entity กับ Dao
```

---

## ขั้นตอนที่ 802: สร้าง Entity (Table)

```kotlin
import androidx.room.*

// @Entity = Table ใน database
@Entity(tableName = "tasks")
data class Task(
    @PrimaryKey(autoGenerate = true)
    val id: Int = 0,

    @ColumnInfo(name = "title")
    val title: String,

    @ColumnInfo(name = "description")
    val description: String = "",

    @ColumnInfo(name = "is_completed")
    val isCompleted: Boolean = false,

    @ColumnInfo(name = "priority")
    val priority: TaskPriority = TaskPriority.MEDIUM,

    @ColumnInfo(name = "created_at")
    val createdAt: Long = System.currentTimeMillis(),

    @ColumnInfo(name = "due_date")
    val dueDate: Long? = null
)

enum class TaskPriority { LOW, MEDIUM, HIGH }

// Entity ที่มี Foreign Key
@Entity(
    tableName = "task_tags",
    foreignKeys = [
        ForeignKey(
            entity = Task::class,
            parentColumns = ["id"],
            childColumns = ["task_id"],
            onDelete = ForeignKey.CASCADE // ลบ task → ลบ tag อัตโนมัติ
        )
    ],
    indices = [Index("task_id")]
)
data class TaskTag(
    @PrimaryKey(autoGenerate = true) val id: Int = 0,
    val taskId: Int,
    val tag: String
)

// TypeConverter สำหรับ type ที่ Room ไม่รู้จัก
class Converters {
    @TypeConverter
    fun fromTaskPriority(priority: TaskPriority): String = priority.name

    @TypeConverter
    fun toTaskPriority(value: String): TaskPriority = TaskPriority.valueOf(value)

    @TypeConverter
    fun fromList(list: List<String>): String = list.joinToString(",")

    @TypeConverter
    fun toList(value: String): List<String> =
        if (value.isEmpty()) emptyList() else value.split(",")
}
```

---

## ขั้นตอนที่ 803: สร้าง Dao (Data Access Object)

```kotlin
import androidx.room.*
import kotlinx.coroutines.flow.Flow

@Dao
interface TaskDao {
    // Query ทั้งหมด - ใช้ Flow เพื่อ observe การเปลี่ยนแปลง
    @Query("SELECT * FROM tasks ORDER BY created_at DESC")
    fun getAllTasks(): Flow<List<Task>>

    // Query ตาม condition
    @Query("SELECT * FROM tasks WHERE is_completed = :isCompleted ORDER BY priority DESC")
    fun getTasksByStatus(isCompleted: Boolean): Flow<List<Task>>

    // Query หา ID เดียว
    @Query("SELECT * FROM tasks WHERE id = :taskId")
    suspend fun getTaskById(taskId: Int): Task?

    // Query นับจำนวน
    @Query("SELECT COUNT(*) FROM tasks WHERE is_completed = 0")
    fun getIncompleteTaskCount(): Flow<Int>

    // Query ค้นหา
    @Query("SELECT * FROM tasks WHERE title LIKE '%' || :query || '%' OR description LIKE '%' || :query || '%'")
    fun searchTasks(query: String): Flow<List<Task>>

    // Insert - คืน id ของ row ที่เพิ่ม
    @Insert(onConflict = OnConflictStrategy.REPLACE)
    suspend fun insertTask(task: Task): Long

    // Insert หลายรายการ
    @Insert(onConflict = OnConflictStrategy.IGNORE)
    suspend fun insertTasks(tasks: List<Task>)

    // Update
    @Update
    suspend fun updateTask(task: Task)

    // Update เฉพาะ field
    @Query("UPDATE tasks SET is_completed = :isCompleted WHERE id = :taskId")
    suspend fun updateTaskStatus(taskId: Int, isCompleted: Boolean)

    // Delete ด้วย object
    @Delete
    suspend fun deleteTask(task: Task)

    // Delete ด้วย condition
    @Query("DELETE FROM tasks WHERE is_completed = 1")
    suspend fun deleteCompletedTasks()

    // Delete ทั้งหมด
    @Query("DELETE FROM tasks")
    suspend fun deleteAllTasks()

    // Transaction - ทำหลาย operation พร้อมกัน
    @Transaction
    suspend fun replaceAllTasks(tasks: List<Task>) {
        deleteAllTasks()
        insertTasks(tasks)
    }
}
```

---

## ขั้นตอนที่ 804: สร้าง Database

```kotlin
import androidx.room.*

@Database(
    entities = [Task::class, TaskTag::class],
    version = 1,
    exportSchema = true // export schema สำหรับ migration testing
)
@TypeConverters(Converters::class)
abstract class AppDatabase : RoomDatabase() {
    abstract fun taskDao(): TaskDao

    companion object {
        @Volatile
        private var INSTANCE: AppDatabase? = null

        fun getDatabase(context: Context): AppDatabase {
            return INSTANCE ?: synchronized(this) {
                val instance = Room.databaseBuilder(
                    context.applicationContext,
                    AppDatabase::class.java,
                    "app_database"
                )
                    .fallbackToDestructiveMigration() // dev เท่านั้น!
                    .build()
                INSTANCE = instance
                instance
            }
        }
    }
}

// Migration - เมื่อเพิ่ม column ใหม่
val MIGRATION_1_2 = object : Migration(1, 2) {
    override fun migrate(database: SupportSQLiteDatabase) {
        database.execSQL(
            "ALTER TABLE tasks ADD COLUMN due_date INTEGER"
        )
    }
}

val MIGRATION_2_3 = object : Migration(2, 3) {
    override fun migrate(database: SupportSQLiteDatabase) {
        database.execSQL(
            "ALTER TABLE tasks ADD COLUMN priority TEXT NOT NULL DEFAULT 'MEDIUM'"
        )
    }
}

// Database พร้อม Migration
fun buildDatabase(context: Context): AppDatabase {
    return Room.databaseBuilder(
        context.applicationContext,
        AppDatabase::class.java,
        "app_database"
    )
        .addMigrations(MIGRATION_1_2, MIGRATION_2_3)
        .build()
}
```

---

## ขั้นตอนที่ 805: ใช้งาน Room ใน ViewModel

```kotlin
import androidx.lifecycle.ViewModel
import androidx.lifecycle.viewModelScope
import kotlinx.coroutines.flow.*
import kotlinx.coroutines.launch

class TaskViewModel(private val taskDao: TaskDao) : ViewModel() {
    // ดึง tasks ทั้งหมด (auto-update เมื่อ database เปลี่ยน)
    val allTasks: StateFlow<List<Task>> = taskDao.getAllTasks()
        .stateIn(
            scope = viewModelScope,
            started = SharingStarted.WhileSubscribed(5000),
            initialValue = emptyList()
        )

    val incompleteCount: StateFlow<Int> = taskDao.getIncompleteTaskCount()
        .stateIn(viewModelScope, SharingStarted.WhileSubscribed(5000), 0)

    fun addTask(title: String, description: String, priority: TaskPriority) {
        viewModelScope.launch {
            val task = Task(title = title, description = description, priority = priority)
            taskDao.insertTask(task)
        }
    }

    fun toggleTaskComplete(task: Task) {
        viewModelScope.launch {
            taskDao.updateTask(task.copy(isCompleted = !task.isCompleted))
        }
    }

    fun deleteTask(task: Task) {
        viewModelScope.launch {
            taskDao.deleteTask(task)
        }
    }

    fun deleteCompleted() {
        viewModelScope.launch {
            taskDao.deleteCompletedTasks()
        }
    }
}

@Composable
fun TaskListScreen(viewModel: TaskViewModel) {
    val tasks by viewModel.allTasks.collectAsStateWithLifecycle()
    val incompleteCount by viewModel.incompleteCount.collectAsStateWithLifecycle()

    Scaffold(
        topBar = {
            TopAppBar(
                title = { Text("Tasks ($incompleteCount ยังเหลือ)") }
            )
        }
    ) { padding ->
        LazyColumn(contentPadding = padding) {
            items(tasks, key = { it.id }) { task ->
                ListItem(
                    headlineContent = { Text(task.title) },
                    leadingContent = {
                        Checkbox(
                            checked = task.isCompleted,
                            onCheckedChange = { viewModel.toggleTaskComplete(task) }
                        )
                    },
                    trailingContent = {
                        IconButton(onClick = { viewModel.deleteTask(task) }) {
                            Icon(Icons.Default.Delete, null)
                        }
                    }
                )
                Divider()
            }
        }
    }
}
```

---

*Part 41 จบแล้ว | ก่อนหน้า: [Part 40](../part40/README.md) | ถัดไป: [Part 42](../part42/README.md)*
