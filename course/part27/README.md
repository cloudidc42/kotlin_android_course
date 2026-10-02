# Part 27: Room Database
## ขั้นตอนที่ 651-675

---

## ขั้นตอนที่ 651: Room คืออะไร?

Room เป็น SQLite abstraction library จาก Jetpack ที่ทำให้การทำงานกับ database ง่ายขึ้น

```
Room Architecture:
┌─────────────────────────────────────────────────┐
│                    App Code                      │
│  (ViewModel, Repository)                         │
└──────────────────┬──────────────────────────────┘
                   │ uses
┌──────────────────▼──────────────────────────────┐
│                  Room                            │
│  ┌─────────┐ ┌────────┐ ┌──────────────────────┐│
│  │ Entity  │ │  DAO   │ │  RoomDatabase        ││
│  │(@Entity)│ │(@Dao)  │ │  (abstract class)    ││
│  └─────────┘ └────────┘ └──────────────────────┘│
└──────────────────┬──────────────────────────────┘
                   │
┌──────────────────▼──────────────────────────────┐
│               SQLite Database                    │
└─────────────────────────────────────────────────┘
```

---

## ขั้นตอนที่ 652: Entity

```kotlin
// app/src/main/java/com/example/app/data/local/entity/

import androidx.room.*

// ============================================
// @Entity - แต่ละ Entity = 1 table ใน database
// ============================================

@Entity(tableName = "users")
data class UserEntity(
    @PrimaryKey(autoGenerate = true)
    val id: Long = 0,
    
    @ColumnInfo(name = "name")
    val name: String,
    
    @ColumnInfo(name = "email")
    val email: String,
    
    @ColumnInfo(name = "phone")
    val phone: String? = null,
    
    @ColumnInfo(name = "created_at")
    val createdAt: Long = System.currentTimeMillis(),
    
    @ColumnInfo(name = "is_active", defaultValue = "1")
    val isActive: Boolean = true
)

// Entity ที่มี Unique constraints
@Entity(
    tableName = "posts",
    indices = [
        Index(value = ["user_id"]),
        Index(value = ["slug"], unique = true)
    ],
    foreignKeys = [
        ForeignKey(
            entity = UserEntity::class,
            parentColumns = ["id"],
            childColumns = ["user_id"],
            onDelete = ForeignKey.CASCADE,  // ลบ user -> ลบ posts ด้วย
            onUpdate = ForeignKey.CASCADE
        )
    ]
)
data class PostEntity(
    @PrimaryKey(autoGenerate = true)
    val id: Long = 0,
    
    @ColumnInfo(name = "user_id")
    val userId: Long,
    
    @ColumnInfo(name = "title")
    val title: String,
    
    @ColumnInfo(name = "content")
    val content: String,
    
    @ColumnInfo(name = "slug")
    val slug: String,
    
    @ColumnInfo(name = "status")
    val status: String = "draft",  // draft, published, archived
    
    @ColumnInfo(name = "view_count")
    val viewCount: Int = 0,
    
    @ColumnInfo(name = "created_at")
    val createdAt: Long = System.currentTimeMillis(),
    
    @ColumnInfo(name = "updated_at")
    val updatedAt: Long = System.currentTimeMillis()
)

// Embedded - ฝัง object ใน Entity
data class Address(
    @ColumnInfo(name = "street") val street: String,
    @ColumnInfo(name = "city") val city: String,
    @ColumnInfo(name = "province") val province: String,
    @ColumnInfo(name = "postal_code") val postalCode: String
)

@Entity(tableName = "customers")
data class CustomerEntity(
    @PrimaryKey(autoGenerate = true) val id: Long = 0,
    val name: String,
    
    @Embedded
    val address: Address
)

// Type Converters - เก็บ custom types
class Converters {
    @TypeConverter
    fun fromStringList(value: String): List<String> {
        return if (value.isEmpty()) emptyList()
        else value.split(",")
    }
    
    @TypeConverter
    fun toStringList(list: List<String>): String {
        return list.joinToString(",")
    }
    
    @TypeConverter
    fun fromTimestamp(value: Long?): java.util.Date? {
        return value?.let { java.util.Date(it) }
    }
    
    @TypeConverter
    fun dateToTimestamp(date: java.util.Date?): Long? {
        return date?.time
    }
}
```

---

## ขั้นตอนที่ 653: DAO (Data Access Object)

```kotlin
// app/src/main/java/com/example/app/data/local/dao/

import androidx.room.*
import kotlinx.coroutines.flow.Flow

// ============================================
// UserDao
// ============================================

@Dao
interface UserDao {
    // ============================================
    // INSERT
    // ============================================
    
    @Insert(onConflict = OnConflictStrategy.REPLACE)
    suspend fun insertUser(user: UserEntity): Long
    
    @Insert(onConflict = OnConflictStrategy.IGNORE)
    suspend fun insertUsers(users: List<UserEntity>): List<Long>
    
    // ============================================
    // UPDATE
    // ============================================
    
    @Update
    suspend fun updateUser(user: UserEntity)
    
    // Custom update
    @Query("UPDATE users SET name = :name, updated_at = :updatedAt WHERE id = :id")
    suspend fun updateUserName(id: Long, name: String, updatedAt: Long = System.currentTimeMillis())
    
    @Query("UPDATE users SET is_active = :isActive WHERE id = :id")
    suspend fun setUserActive(id: Long, isActive: Boolean)
    
    // ============================================
    // DELETE
    // ============================================
    
    @Delete
    suspend fun deleteUser(user: UserEntity)
    
    @Query("DELETE FROM users WHERE id = :id")
    suspend fun deleteUserById(id: Long)
    
    @Query("DELETE FROM users WHERE is_active = 0")
    suspend fun deleteInactiveUsers(): Int  // return จำนวนแถวที่ลบ
    
    @Query("DELETE FROM users")
    suspend fun deleteAllUsers()
    
    // ============================================
    // SELECT - suspend functions
    // ============================================
    
    @Query("SELECT * FROM users WHERE id = :id")
    suspend fun getUserById(id: Long): UserEntity?
    
    @Query("SELECT * FROM users WHERE email = :email LIMIT 1")
    suspend fun getUserByEmail(email: String): UserEntity?
    
    @Query("SELECT * FROM users ORDER BY created_at DESC")
    suspend fun getAllUsers(): List<UserEntity>
    
    @Query("SELECT COUNT(*) FROM users")
    suspend fun getUserCount(): Int
    
    // ============================================
    // SELECT - Flow (Reactive)
    // ============================================
    
    @Query("SELECT * FROM users WHERE id = :id")
    fun observeUser(id: Long): Flow<UserEntity?>
    
    @Query("SELECT * FROM users WHERE is_active = 1 ORDER BY name ASC")
    fun observeActiveUsers(): Flow<List<UserEntity>>
    
    @Query("SELECT * FROM users ORDER BY created_at DESC LIMIT :limit OFFSET :offset")
    fun observeUsersPaged(limit: Int = 20, offset: Int = 0): Flow<List<UserEntity>>
    
    // ============================================
    // Search
    // ============================================
    
    @Query("""
        SELECT * FROM users 
        WHERE name LIKE '%' || :query || '%' 
           OR email LIKE '%' || :query || '%'
        ORDER BY name ASC
    """)
    fun searchUsers(query: String): Flow<List<UserEntity>>
    
    // ============================================
    // Transactions
    // ============================================
    
    @Transaction
    suspend fun replaceUser(user: UserEntity) {
        deleteUserById(user.id)
        insertUser(user)
    }
}

// ============================================
// PostDao
// ============================================

@Dao
interface PostDao {
    @Insert(onConflict = OnConflictStrategy.REPLACE)
    suspend fun insertPost(post: PostEntity): Long
    
    @Insert(onConflict = OnConflictStrategy.REPLACE)
    suspend fun insertPosts(posts: List<PostEntity>)
    
    @Update
    suspend fun updatePost(post: PostEntity)
    
    @Delete
    suspend fun deletePost(post: PostEntity)
    
    @Query("SELECT * FROM posts WHERE id = :id")
    suspend fun getPostById(id: Long): PostEntity?
    
    @Query("SELECT * FROM posts WHERE user_id = :userId ORDER BY created_at DESC")
    fun observePostsByUser(userId: Long): Flow<List<PostEntity>>
    
    @Query("SELECT * FROM posts WHERE status = 'published' ORDER BY created_at DESC")
    fun observePublishedPosts(): Flow<List<PostEntity>>
    
    @Query("UPDATE posts SET view_count = view_count + 1 WHERE id = :postId")
    suspend fun incrementViewCount(postId: Long)
    
    // Relation - Join query
    @Query("""
        SELECT p.*, u.name as author_name, u.email as author_email
        FROM posts p
        INNER JOIN users u ON p.user_id = u.id
        WHERE p.status = 'published'
        ORDER BY p.created_at DESC
    """)
    fun observePublishedPostsWithAuthors(): Flow<List<PostWithAuthor>>
}

// ============================================
// Relation Classes
// ============================================

data class PostWithAuthor(
    @Embedded val post: PostEntity,
    @ColumnInfo(name = "author_name") val authorName: String,
    @ColumnInfo(name = "author_email") val authorEmail: String
)

// @Relation - One-to-Many
data class UserWithPosts(
    @Embedded val user: UserEntity,
    @Relation(
        parentColumn = "id",
        entityColumn = "user_id"
    )
    val posts: List<PostEntity>
)

@Dao
interface UserWithPostsDao {
    @Transaction
    @Query("SELECT * FROM users WHERE id = :userId")
    fun getUserWithPosts(userId: Long): Flow<UserWithPosts?>
    
    @Transaction
    @Query("SELECT * FROM users ORDER BY name ASC")
    fun getAllUsersWithPosts(): Flow<List<UserWithPosts>>
}
```

---

## ขั้นตอนที่ 654: RoomDatabase

```kotlin
// app/src/main/java/com/example/app/data/local/

import androidx.room.*
import androidx.sqlite.db.SupportSQLiteDatabase

@Database(
    entities = [
        UserEntity::class,
        PostEntity::class,
        CustomerEntity::class
    ],
    version = 3,  // เพิ่ม version ทุกครั้งที่ schema เปลี่ยน
    exportSchema = true
)
@TypeConverters(Converters::class)
abstract class AppDatabase : RoomDatabase() {
    
    abstract fun userDao(): UserDao
    abstract fun postDao(): PostDao
    abstract fun userWithPostsDao(): UserWithPostsDao
    
    companion object {
        @Volatile
        private var INSTANCE: AppDatabase? = null
        
        fun getInstance(context: Context): AppDatabase {
            return INSTANCE ?: synchronized(this) {
                val instance = Room.databaseBuilder(
                    context.applicationContext,
                    AppDatabase::class.java,
                    "app_database"
                )
                .addMigrations(MIGRATION_1_2, MIGRATION_2_3)
                .addCallback(object : RoomDatabase.Callback() {
                    override fun onCreate(db: SupportSQLiteDatabase) {
                        super.onCreate(db)
                        // pre-populate database
                        CoroutineScope(Dispatchers.IO).launch {
                            prepopulateDatabase(getInstance(context))
                        }
                    }
                })
                .build()
                INSTANCE = instance
                instance
            }
        }
        
        private suspend fun prepopulateDatabase(db: AppDatabase) {
            db.userDao().insertUsers(
                listOf(
                    UserEntity(name = "Admin", email = "admin@example.com"),
                    UserEntity(name = "Test User", email = "test@example.com")
                )
            )
        }
    }
}

// ============================================
// Migrations
// ============================================

val MIGRATION_1_2 = object : Migration(1, 2) {
    override fun migrate(database: SupportSQLiteDatabase) {
        // เพิ่ม column ใหม่
        database.execSQL("ALTER TABLE users ADD COLUMN phone TEXT")
    }
}

val MIGRATION_2_3 = object : Migration(2, 3) {
    override fun migrate(database: SupportSQLiteDatabase) {
        // สร้าง table ใหม่
        database.execSQL("""
            CREATE TABLE IF NOT EXISTS customers (
                id INTEGER PRIMARY KEY AUTOINCREMENT NOT NULL,
                name TEXT NOT NULL,
                street TEXT NOT NULL,
                city TEXT NOT NULL,
                province TEXT NOT NULL,
                postal_code TEXT NOT NULL
            )
        """)
    }
}
```

---

## ขั้นตอนที่ 655: Repository Pattern กับ Room

```kotlin
// app/src/main/java/com/example/app/data/repository/

import kotlinx.coroutines.flow.Flow
import kotlinx.coroutines.flow.map

// Domain Model (แยกจาก Entity)
data class User(
    val id: Long = 0,
    val name: String,
    val email: String,
    val phone: String? = null,
    val isActive: Boolean = true
)

// Mapper functions
fun UserEntity.toDomain(): User = User(
    id = id,
    name = name,
    email = email,
    phone = phone,
    isActive = isActive
)

fun User.toEntity(): UserEntity = UserEntity(
    id = id,
    name = name,
    email = email,
    phone = phone,
    isActive = isActive
)

// Repository Interface
interface UserRepository {
    suspend fun saveUser(user: User): Long
    suspend fun updateUser(user: User)
    suspend fun deleteUser(id: Long)
    suspend fun getUser(id: Long): User?
    suspend fun getAllUsers(): List<User>
    fun observeUser(id: Long): Flow<User?>
    fun observeActiveUsers(): Flow<List<User>>
    fun searchUsers(query: String): Flow<List<User>>
}

// Repository Implementation
class UserRepositoryImpl(
    private val userDao: UserDao,
    private val ioDispatcher: CoroutineDispatcher = Dispatchers.IO
) : UserRepository {
    
    override suspend fun saveUser(user: User): Long {
        return withContext(ioDispatcher) {
            userDao.insertUser(user.toEntity())
        }
    }
    
    override suspend fun updateUser(user: User) {
        withContext(ioDispatcher) {
            userDao.updateUser(user.toEntity())
        }
    }
    
    override suspend fun deleteUser(id: Long) {
        withContext(ioDispatcher) {
            userDao.deleteUserById(id)
        }
    }
    
    override suspend fun getUser(id: Long): User? {
        return withContext(ioDispatcher) {
            userDao.getUserById(id)?.toDomain()
        }
    }
    
    override suspend fun getAllUsers(): List<User> {
        return withContext(ioDispatcher) {
            userDao.getAllUsers().map { it.toDomain() }
        }
    }
    
    override fun observeUser(id: Long): Flow<User?> {
        return userDao.observeUser(id).map { it?.toDomain() }
    }
    
    override fun observeActiveUsers(): Flow<List<User>> {
        return userDao.observeActiveUsers().map { entities ->
            entities.map { it.toDomain() }
        }
    }
    
    override fun searchUsers(query: String): Flow<List<User>> {
        return userDao.searchUsers(query).map { entities ->
            entities.map { it.toDomain() }
        }
    }
}
```

---

## ขั้นตอนที่ 656: ViewModel กับ Room

```kotlin
// app/src/main/java/com/example/app/ui/users/

@HiltViewModel
class UsersViewModel @Inject constructor(
    private val userRepository: UserRepository
) : ViewModel() {
    
    private val _searchQuery = MutableStateFlow("")
    
    val users: StateFlow<List<User>> = _searchQuery
        .debounce(300)
        .flatMapLatest { query ->
            if (query.isBlank()) userRepository.observeActiveUsers()
            else userRepository.searchUsers(query)
        }
        .stateIn(
            scope = viewModelScope,
            started = SharingStarted.WhileSubscribed(5000),
            initialValue = emptyList()
        )
    
    private val _uiState = MutableStateFlow<UiState>(UiState.Idle)
    val uiState: StateFlow<UiState> = _uiState.asStateFlow()
    
    fun search(query: String) { _searchQuery.value = query }
    
    fun saveUser(name: String, email: String) {
        viewModelScope.launch {
            _uiState.value = UiState.Loading
            try {
                val user = User(name = name, email = email)
                userRepository.saveUser(user)
                _uiState.value = UiState.Success("บันทึกสำเร็จ!")
            } catch (e: Exception) {
                _uiState.value = UiState.Error(e.message ?: "เกิดข้อผิดพลาด")
            }
        }
    }
    
    fun deleteUser(id: Long) {
        viewModelScope.launch {
            try {
                userRepository.deleteUser(id)
            } catch (e: Exception) {
                _uiState.value = UiState.Error("ลบไม่สำเร็จ")
            }
        }
    }
    
    sealed class UiState {
        object Idle : UiState()
        object Loading : UiState()
        data class Success(val message: String) : UiState()
        data class Error(val message: String) : UiState()
    }
}

// Screen
@Composable
fun UsersScreen(viewModel: UsersViewModel = hiltViewModel()) {
    val users by viewModel.users.collectAsStateWithLifecycle()
    val uiState by viewModel.uiState.collectAsStateWithLifecycle()
    
    var showAddDialog by remember { mutableStateOf(false) }
    var searchQuery by remember { mutableStateOf("") }
    
    LaunchedEffect(searchQuery) {
        viewModel.search(searchQuery)
    }
    
    Scaffold(
        topBar = {
            TopAppBar(title = { Text("ผู้ใช้งาน") })
        },
        floatingActionButton = {
            FloatingActionButton(onClick = { showAddDialog = true }) {
                Icon(Icons.Default.Add, null)
            }
        }
    ) { padding ->
        Column(
            modifier = Modifier
                .fillMaxSize()
                .padding(padding)
        ) {
            // Search bar
            OutlinedTextField(
                value = searchQuery,
                onValueChange = { searchQuery = it },
                label = { Text("ค้นหา") },
                modifier = Modifier
                    .fillMaxWidth()
                    .padding(16.dp),
                leadingIcon = { Icon(Icons.Default.Search, null) }
            )
            
            // Users list
            LazyColumn {
                items(users, key = { it.id }) { user ->
                    UserListItem(
                        user = user,
                        onDelete = { viewModel.deleteUser(user.id) }
                    )
                }
            }
        }
        
        // Add user dialog
        if (showAddDialog) {
            AddUserDialog(
                onDismiss = { showAddDialog = false },
                onConfirm = { name, email ->
                    viewModel.saveUser(name, email)
                    showAddDialog = false
                }
            )
        }
        
        // Handle UI state
        when (val state = uiState) {
            is UsersViewModel.UiState.Loading -> {
                Box(modifier = Modifier.fillMaxSize()) {
                    CircularProgressIndicator(modifier = Modifier.align(Alignment.Center))
                }
            }
            is UsersViewModel.UiState.Error -> {
                // Show error snackbar
            }
            else -> {}
        }
    }
}

@Composable
fun UserListItem(user: User, onDelete: () -> Unit) {
    ListItem(
        headlineContent = { Text(user.name) },
        supportingContent = { Text(user.email) },
        trailingContent = {
            IconButton(onClick = onDelete) {
                Icon(
                    Icons.Default.Delete,
                    contentDescription = "Delete",
                    tint = MaterialTheme.colorScheme.error
                )
            }
        }
    )
    HorizontalDivider()
}

@Composable
fun AddUserDialog(
    onDismiss: () -> Unit,
    onConfirm: (name: String, email: String) -> Unit
) {
    var name by remember { mutableStateOf("") }
    var email by remember { mutableStateOf("") }
    
    AlertDialog(
        onDismissRequest = onDismiss,
        title = { Text("เพิ่มผู้ใช้ใหม่") },
        text = {
            Column(verticalArrangement = Arrangement.spacedBy(8.dp)) {
                OutlinedTextField(
                    value = name,
                    onValueChange = { name = it },
                    label = { Text("ชื่อ") },
                    modifier = Modifier.fillMaxWidth()
                )
                OutlinedTextField(
                    value = email,
                    onValueChange = { email = it },
                    label = { Text("Email") },
                    modifier = Modifier.fillMaxWidth()
                )
            }
        },
        confirmButton = {
            Button(
                onClick = { onConfirm(name, email) },
                enabled = name.isNotBlank() && email.isNotBlank()
            ) { Text("บันทึก") }
        },
        dismissButton = {
            TextButton(onClick = onDismiss) { Text("ยกเลิก") }
        }
    )
}
```

---

## ขั้นตอนที่ 657: Hilt DI กับ Room

```kotlin
// app/src/main/java/com/example/app/di/

@Module
@InstallIn(SingletonComponent::class)
object DatabaseModule {
    
    @Provides
    @Singleton
    fun provideDatabase(@ApplicationContext context: Context): AppDatabase {
        return Room.databaseBuilder(
            context,
            AppDatabase::class.java,
            "app_database"
        )
        .addMigrations(MIGRATION_1_2, MIGRATION_2_3)
        .build()
    }
    
    @Provides
    fun provideUserDao(database: AppDatabase): UserDao = database.userDao()
    
    @Provides
    fun providePostDao(database: AppDatabase): PostDao = database.postDao()
}

@Module
@InstallIn(SingletonComponent::class)
object RepositoryModule {
    
    @Provides
    @Singleton
    fun provideUserRepository(
        userDao: UserDao,
        @IoDispatcher ioDispatcher: CoroutineDispatcher
    ): UserRepository = UserRepositoryImpl(userDao, ioDispatcher)
}

@Module
@InstallIn(SingletonComponent::class)
object DispatcherModule {
    
    @Provides
    @DefaultDispatcher
    fun provideDefaultDispatcher(): CoroutineDispatcher = Dispatchers.Default
    
    @Provides
    @IoDispatcher
    fun provideIoDispatcher(): CoroutineDispatcher = Dispatchers.IO
    
    @Provides
    @MainDispatcher
    fun provideMainDispatcher(): CoroutineDispatcher = Dispatchers.Main
}

// Qualifier annotations
@Qualifier
@Retention(AnnotationRetention.BINARY)
annotation class DefaultDispatcher

@Qualifier
@Retention(AnnotationRetention.BINARY)
annotation class IoDispatcher

@Qualifier
@Retention(AnnotationRetention.BINARY)
annotation class MainDispatcher
```

---

## แบบฝึกหัด Part 27

```kotlin
// แบบฝึกหัดที่ 1: Note App Database
// สร้าง Room database สำหรับ Note App

@Entity(tableName = "notes")
data class NoteEntity(
    @PrimaryKey(autoGenerate = true) val id: Long = 0,
    val title: String,
    val content: String,
    val color: Int = 0xFFFFFF,  // background color
    val isPinned: Boolean = false,
    val createdAt: Long = System.currentTimeMillis(),
    val updatedAt: Long = System.currentTimeMillis()
)

// TODO: สร้าง NoteDao ที่มี:
// - insertNote, updateNote, deleteNote
// - getNoteById, observeAllNotes (Flow)
// - observePinnedNotes (Flow)
// - searchNotes(query) (Flow)
// - updatePinStatus(id, isPinned)

@Dao
interface NoteDao {
    // TODO: implement
}

// TODO: สร้าง AppDatabase ที่รวม NoteEntity

// แบบฝึกหัดที่ 2: Favorite Products
// เพิ่ม functionality ให้ User สามารถ favorite product ได้

@Entity(
    tableName = "user_favorites",
    primaryKeys = ["user_id", "product_id"]
)
data class UserFavoriteEntity(
    @ColumnInfo(name = "user_id") val userId: Long,
    @ColumnInfo(name = "product_id") val productId: Long,
    val savedAt: Long = System.currentTimeMillis()
)

// TODO: สร้าง FavoriteDao
// TODO: สร้าง FavoriteRepository
```

---

*Part 27 จบแล้ว | ก่อนหน้า: [Part 26](../part26/README.md) | ถัดไป: [Part 28](../part28/README.md)*
