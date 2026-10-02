# Part 74: Advanced Room Database
## ขั้นตอนที่ 1326-1350

---

## ขั้นตอนที่ 1326: Room FTS (Full-Text Search)

```kotlin
// Full-Text Search ด้วย Room + SQLite FTS4/FTS5
// ใช้สำหรับ search ข้อความเร็วมาก

@Entity(tableName = "notes_fts")
@Fts4(contentEntity = NoteEntity::class)  // หรือ @Fts5
data class NoteFts(
    @PrimaryKey @ColumnInfo(name = "rowid") val rowId: Int = 0,
    val title: String,
    val content: String
)

@Entity(tableName = "notes")
data class NoteEntity(
    @PrimaryKey(autoGenerate = true) val id: Long = 0,
    val title: String,
    val content: String,
    val createdAt: Long = System.currentTimeMillis(),
    val tags: String = ""  // comma-separated
)

@Dao
interface NoteDao {
    
    @Insert(onConflict = OnConflictStrategy.REPLACE)
    suspend fun insert(note: NoteEntity): Long
    
    @Update
    suspend fun update(note: NoteEntity)
    
    @Delete
    suspend fun delete(note: NoteEntity)
    
    @Query("SELECT * FROM notes ORDER BY createdAt DESC")
    fun observeAll(): Flow<List<NoteEntity>>
    
    // FTS Search - ค้นหาใน title และ content
    @Query("""
        SELECT notes.* FROM notes
        INNER JOIN notes_fts ON notes.rowid = notes_fts.rowid
        WHERE notes_fts MATCH :query
        ORDER BY rank
    """)
    fun search(query: String): Flow<List<NoteEntity>>
    
    // FTS Search with snippet (แสดง context รอบคำที่ค้นหา)
    @Query("""
        SELECT notes.*, snippet(notes_fts, 1, '<b>', '</b>', '...', 20) AS snippet
        FROM notes
        INNER JOIN notes_fts ON notes.rowid = notes_fts.rowid
        WHERE notes_fts MATCH :query
    """)
    fun searchWithSnippet(query: String): Flow<List<NoteWithSnippet>>
    
    @Query("SELECT COUNT(*) FROM notes")
    fun count(): Flow<Int>
}

data class NoteWithSnippet(
    @Embedded val note: NoteEntity,
    val snippet: String
)

// FTS query formatting
fun formatFtsQuery(input: String): String {
    // Escape special characters and format for FTS
    val escaped = input.trim()
        .replace("\"", "\"\"")
    return "\"$escaped*\""  // prefix match
}
```

---

## ขั้นตอนที่ 1327: Room Relations

```kotlin
// ============================================
// One-to-Many Relationship
// ============================================

@Entity(tableName = "authors")
data class AuthorEntity(
    @PrimaryKey val id: Long,
    val name: String,
    val email: String
)

@Entity(
    tableName = "books",
    foreignKeys = [
        ForeignKey(
            entity = AuthorEntity::class,
            parentColumns = ["id"],
            childColumns = ["authorId"],
            onDelete = ForeignKey.CASCADE  // ลบ books เมื่อ author ถูกลบ
        )
    ],
    indices = [Index("authorId")]
)
data class BookEntity(
    @PrimaryKey val id: Long,
    val title: String,
    val authorId: Long,
    val publishYear: Int
)

data class AuthorWithBooks(
    @Embedded val author: AuthorEntity,
    @Relation(
        parentColumn = "id",
        entityColumn = "authorId"
    )
    val books: List<BookEntity>
)

// ============================================
// Many-to-Many Relationship
// ============================================

@Entity(tableName = "tags")
data class TagEntity(
    @PrimaryKey val id: Long,
    val name: String,
    val color: String
)

@Entity(
    tableName = "note_tag_cross_ref",
    primaryKeys = ["noteId", "tagId"]
)
data class NoteTagCrossRef(
    val noteId: Long,
    val tagId: Long
)

data class NoteWithTags(
    @Embedded val note: NoteEntity,
    @Relation(
        parentColumn = "id",
        entityColumn = "id",
        associateBy = Junction(NoteTagCrossRef::class)
    )
    val tags: List<TagEntity>
)

data class TagWithNotes(
    @Embedded val tag: TagEntity,
    @Relation(
        parentColumn = "id",
        entityColumn = "id",
        associateBy = Junction(NoteTagCrossRef::class)
    )
    val notes: List<NoteEntity>
)

@Dao
interface NoteWithTagsDao {
    
    @Transaction
    @Query("SELECT * FROM notes WHERE id = :noteId")
    suspend fun getNoteWithTags(noteId: Long): NoteWithTags?
    
    @Transaction
    @Query("SELECT * FROM notes")
    fun observeNotesWithTags(): Flow<List<NoteWithTags>>
    
    @Insert(onConflict = OnConflictStrategy.IGNORE)
    suspend fun insertCrossRef(crossRef: NoteTagCrossRef)
    
    @Delete
    suspend fun deleteCrossRef(crossRef: NoteTagCrossRef)
    
    @Transaction
    suspend fun setNoteTags(noteId: Long, tagIds: List<Long>) {
        // Delete existing
        deleteTagsForNote(noteId)
        // Insert new
        tagIds.forEach { tagId ->
            insertCrossRef(NoteTagCrossRef(noteId, tagId))
        }
    }
    
    @Query("DELETE FROM note_tag_cross_ref WHERE noteId = :noteId")
    suspend fun deleteTagsForNote(noteId: Long)
}
```

---

## ขั้นตอนที่ 1328: Room Migrations

```kotlin
// ============================================
// Database Migration
// ============================================

@Database(
    entities = [NoteEntity::class, NoteFts::class, TagEntity::class, NoteTagCrossRef::class],
    version = 3,
    exportSchema = true  // เก็บ schema history ใน assets/
)
abstract class AppDatabase : RoomDatabase() {
    abstract fun noteDao(): NoteDao
    abstract fun tagDao(): TagDao
    abstract fun noteWithTagsDao(): NoteWithTagsDao
}

// Migration 1 → 2: เพิ่ม column "color" ใน notes
val MIGRATION_1_2 = object : Migration(1, 2) {
    override fun migrate(db: SupportSQLiteDatabase) {
        db.execSQL("ALTER TABLE notes ADD COLUMN color TEXT NOT NULL DEFAULT '#FFFFFF'")
    }
}

// Migration 2 → 3: สร้าง tags table และ cross-ref
val MIGRATION_2_3 = object : Migration(2, 3) {
    override fun migrate(db: SupportSQLiteDatabase) {
        // Create tags table
        db.execSQL("""
            CREATE TABLE IF NOT EXISTS tags (
                id INTEGER PRIMARY KEY NOT NULL,
                name TEXT NOT NULL,
                color TEXT NOT NULL
            )
        """)
        
        // Create cross-ref table
        db.execSQL("""
            CREATE TABLE IF NOT EXISTS note_tag_cross_ref (
                noteId INTEGER NOT NULL,
                tagId INTEGER NOT NULL,
                PRIMARY KEY(noteId, tagId),
                FOREIGN KEY(noteId) REFERENCES notes(id) ON DELETE CASCADE,
                FOREIGN KEY(tagId) REFERENCES tags(id) ON DELETE CASCADE
            )
        """)
        
        db.execSQL("CREATE INDEX IF NOT EXISTS index_note_tag_cross_ref_tagId ON note_tag_cross_ref(tagId)")
    }
}

// Provide database with migrations
@Module
@InstallIn(SingletonComponent::class)
object DatabaseModule {
    
    @Provides
    @Singleton
    fun provideDatabase(@ApplicationContext context: Context): AppDatabase {
        return Room.databaseBuilder(context, AppDatabase::class.java, "app.db")
            .addMigrations(MIGRATION_1_2, MIGRATION_2_3)
            .fallbackToDestructiveMigration()  // เฉพาะ dev - ลบข้อมูลถ้า migrate ไม่ได้
            .build()
    }
}
```

---

## ขั้นตอนที่ 1329: Room Type Converters

```kotlin
// ============================================
// Custom Type Converters
// ============================================

class Converters {
    
    private val gson = Gson()
    
    @TypeConverter
    fun fromStringList(value: List<String>?): String? {
        return value?.let { gson.toJson(it) }
    }
    
    @TypeConverter
    fun toStringList(value: String?): List<String>? {
        return value?.let {
            gson.fromJson(it, Array<String>::class.java)?.toList()
        }
    }
    
    @TypeConverter
    fun fromDate(date: LocalDate?): String? = date?.toString()
    
    @TypeConverter
    fun toDate(value: String?): LocalDate? = value?.let { LocalDate.parse(it) }
    
    @TypeConverter
    fun fromInstant(instant: Instant?): Long? = instant?.toEpochMilli()
    
    @TypeConverter
    fun toInstant(value: Long?): Instant? = value?.let { Instant.ofEpochMilli(it) }
    
    @TypeConverter
    fun fromLocation(location: LatLng?): String? {
        return location?.let { "${it.latitude},${it.longitude}" }
    }
    
    @TypeConverter
    fun toLocation(value: String?): LatLng? {
        return value?.split(",")?.let { parts ->
            if (parts.size == 2) {
                LatLng(parts[0].toDouble(), parts[1].toDouble())
            } else null
        }
    }
}

@Database(
    entities = [NoteEntity::class],
    version = 1
)
@TypeConverters(Converters::class)  // Register converters
abstract class AppDatabase : RoomDatabase() {
    abstract fun noteDao(): NoteDao
}
```

---

## ขั้นตอนที่ 1330: Multi-Database Architecture

```kotlin
// ============================================
// Multiple Databases (แยก concern)
// ============================================

// Database 1: User data (sensitive)
@Database(entities = [UserEntity::class, SessionEntity::class], version = 1)
abstract class UserDatabase : RoomDatabase() {
    abstract fun userDao(): UserDao
    abstract fun sessionDao(): SessionDao
}

// Database 2: App content
@Database(entities = [NoteEntity::class, TagEntity::class], version = 1)
abstract class ContentDatabase : RoomDatabase() {
    abstract fun noteDao(): NoteDao
    abstract fun tagDao(): TagDao
}

// Database 3: Analytics (write-heavy, don't lock content)
@Database(entities = [EventEntity::class], version = 1)
abstract class AnalyticsDatabase : RoomDatabase() {
    abstract fun eventDao(): EventDao
}

@Module
@InstallIn(SingletonComponent::class)
object DatabaseModule {
    
    @Provides
    @Singleton
    @Named("user_db")
    fun provideUserDatabase(@ApplicationContext context: Context): UserDatabase {
        return Room.databaseBuilder(context, UserDatabase::class.java, "user.db")
            .build()
    }
    
    @Provides
    @Singleton
    @Named("content_db")
    fun provideContentDatabase(@ApplicationContext context: Context): ContentDatabase {
        return Room.databaseBuilder(context, ContentDatabase::class.java, "content.db")
            .addMigrations(/* migrations */)
            .build()
    }
    
    @Provides
    @Singleton
    @Named("analytics_db")
    fun provideAnalyticsDatabase(@ApplicationContext context: Context): AnalyticsDatabase {
        return Room.databaseBuilder(context, AnalyticsDatabase::class.java, "analytics.db")
            .setJournalMode(RoomDatabase.JournalMode.WRITE_AHEAD_LOGGING)  // Better concurrent writes
            .build()
    }
}
```

---

## แบบฝึกหัด Part 74

```kotlin
// แบบฝึกหัด: Note App ด้วย Advanced Room

// สร้าง Note App ที่:
// 1. CRUD notes พร้อม tags
// 2. FTS search ที่ debounce 300ms
// 3. Filter by tags
// 4. Sort by: date, title, tag
// 5. Soft delete (isDeleted flag + Trash view)
// 6. Export notes เป็น JSON

data class NoteFilter(
    val query: String = "",
    val tagIds: Set<Long> = emptySet(),
    val sortBy: SortBy = SortBy.DATE_DESC,
    val showDeleted: Boolean = false
)

enum class SortBy {
    DATE_DESC, DATE_ASC, TITLE_ASC, TITLE_DESC
}

@HiltViewModel
class NoteListViewModel @Inject constructor(
    private val noteDao: NoteDao
) : ViewModel() {
    
    private val _filter = MutableStateFlow(NoteFilter())
    
    val notes: Flow<List<NoteWithTags>> = _filter
        .debounce(300)
        .flatMapLatest { filter ->
            // TODO: implement filtered query
            noteDao.observeNotesWithTags()
        }
    
    fun updateFilter(filter: NoteFilter) { _filter.value = filter }
}
```

---

*Part 74 จบแล้ว | ก่อนหน้า: [Part 73](../part73/README.md) | ถัดไป: [Part 75](../part75/README.md)*
