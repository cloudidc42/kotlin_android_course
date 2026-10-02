# Part 42: Android - Room กับ Flow & ViewModel
## ขั้นตอนที่ 826-850

---

## ขั้นตอนที่ 826: Repository Pattern กับ Room

Repository เป็น abstraction layer ระหว่าง data source (Room, API) กับ ViewModel

```kotlin
// Repository Interface
interface NoteRepository {
    fun getAllNotes(): Flow<List<Note>>
    fun getNoteById(id: Int): Flow<Note?>
    fun searchNotes(query: String): Flow<List<Note>>
    suspend fun insertNote(note: Note): Long
    suspend fun updateNote(note: Note)
    suspend fun deleteNote(note: Note)
    suspend fun deleteAllNotes()
}

// Entity
@Entity(tableName = "notes")
data class Note(
    @PrimaryKey(autoGenerate = true) val id: Int = 0,
    val title: String,
    val content: String,
    val color: Int = 0xFFFFFF,
    val isPinned: Boolean = false,
    val createdAt: Long = System.currentTimeMillis(),
    val updatedAt: Long = System.currentTimeMillis()
)

// Dao
@Dao
interface NoteDao {
    @Query("SELECT * FROM notes ORDER BY is_pinned DESC, updated_at DESC")
    fun getAllNotes(): Flow<List<Note>>

    @Query("SELECT * FROM notes WHERE id = :id")
    fun getNoteById(id: Int): Flow<Note?>

    @Query("""SELECT * FROM notes WHERE 
        title LIKE '%' || :query || '%' OR 
        content LIKE '%' || :query || '%'
        ORDER BY updated_at DESC""")
    fun searchNotes(query: String): Flow<List<Note>>

    @Insert(onConflict = OnConflictStrategy.REPLACE)
    suspend fun insert(note: Note): Long

    @Update
    suspend fun update(note: Note)

    @Delete
    suspend fun delete(note: Note)

    @Query("DELETE FROM notes")
    suspend fun deleteAll()
}

// Repository Implementation
class NoteRepositoryImpl(private val noteDao: NoteDao) : NoteRepository {
    override fun getAllNotes(): Flow<List<Note>> = noteDao.getAllNotes()
    override fun getNoteById(id: Int): Flow<Note?> = noteDao.getNoteById(id)
    override fun searchNotes(query: String): Flow<List<Note>> = noteDao.searchNotes(query)
    override suspend fun insertNote(note: Note): Long = noteDao.insert(note)
    override suspend fun updateNote(note: Note) = noteDao.update(note.copy(updatedAt = System.currentTimeMillis()))
    override suspend fun deleteNote(note: Note) = noteDao.delete(note)
    override suspend fun deleteAllNotes() = noteDao.deleteAll()
}
```

---

## ขั้นตอนที่ 827: ViewModel ที่ซับซ้อนกับ Room Flow

```kotlin
import androidx.lifecycle.*
import kotlinx.coroutines.ExperimentalCoroutinesApi
import kotlinx.coroutines.FlowPreview
import kotlinx.coroutines.flow.*
import kotlinx.coroutines.launch

data class NoteListUiState(
    val notes: List<Note> = emptyList(),
    val searchQuery: String = "",
    val isLoading: Boolean = false,
    val selectedNotes: Set<Int> = emptySet(),
    val sortBy: SortBy = SortBy.DATE_UPDATED
)

enum class SortBy { DATE_UPDATED, DATE_CREATED, TITLE }

@OptIn(ExperimentalCoroutinesApi::class, FlowPreview::class)
class NoteListViewModel(private val repository: NoteRepository) : ViewModel() {

    private val _searchQuery = MutableStateFlow("")
    private val _sortBy = MutableStateFlow(SortBy.DATE_UPDATED)
    private val _selectedNotes = MutableStateFlow<Set<Int>>(emptySet())

    // combine หลาย flow เป็นหนึ่ง
    val uiState: StateFlow<NoteListUiState> = combine(
        _searchQuery
            .debounce(300) // รอ 300ms หลังจาก type หยุด
            .distinctUntilChanged()
            .flatMapLatest { query ->
                if (query.isEmpty()) repository.getAllNotes()
                else repository.searchNotes(query)
            },
        _sortBy,
        _selectedNotes
    ) { notes, sortBy, selected ->
        val sortedNotes = when (sortBy) {
            SortBy.DATE_UPDATED -> notes.sortedByDescending { it.updatedAt }
            SortBy.DATE_CREATED -> notes.sortedByDescending { it.createdAt }
            SortBy.TITLE -> notes.sortedBy { it.title }
        }
        NoteListUiState(
            notes = sortedNotes,
            searchQuery = _searchQuery.value,
            selectedNotes = selected,
            sortBy = sortBy
        )
    }.stateIn(
        scope = viewModelScope,
        started = SharingStarted.WhileSubscribed(5000),
        initialValue = NoteListUiState(isLoading = true)
    )

    fun onSearchQueryChange(query: String) {
        _searchQuery.value = query
    }

    fun onSortByChange(sortBy: SortBy) {
        _sortBy.value = sortBy
    }

    fun toggleNoteSelection(noteId: Int) {
        _selectedNotes.update { selected ->
            if (noteId in selected) selected - noteId else selected + noteId
        }
    }

    fun deleteSelectedNotes() {
        viewModelScope.launch {
            val selected = _selectedNotes.value.toList()
            selected.forEach { id ->
                val note = repository.getNoteById(id).first()
                note?.let { repository.deleteNote(it) }
            }
            _selectedNotes.value = emptySet()
        }
    }

    fun pinNote(note: Note) {
        viewModelScope.launch {
            repository.updateNote(note.copy(isPinned = !note.isPinned))
        }
    }
}
```

---

## ขั้นตอนที่ 828: Note Editor Screen

```kotlin
import androidx.lifecycle.SavedStateHandle
import kotlinx.coroutines.flow.*
import kotlinx.coroutines.launch

// ViewModel สำหรับหน้าแก้ไข Note
class NoteEditViewModel(
    private val repository: NoteRepository,
    savedStateHandle: SavedStateHandle
) : ViewModel() {

    private val noteId: Int = savedStateHandle.get<Int>("noteId") ?: -1
    private val _isNew = noteId == -1

    private val _title = MutableStateFlow("")
    private val _content = MutableStateFlow("")
    private val _color = MutableStateFlow(0xFFFFFF)

    val title: StateFlow<String> = _title.asStateFlow()
    val content: StateFlow<String> = _content.asStateFlow()
    val color: StateFlow<Int> = _color.asStateFlow()

    val hasChanges: StateFlow<Boolean> = combine(_title, _content) { t, c ->
        t.isNotBlank() || c.isNotBlank()
    }.stateIn(viewModelScope, SharingStarted.Eagerly, false)

    init {
        if (!_isNew) {
            viewModelScope.launch {
                repository.getNoteById(noteId).first()?.let { note ->
                    _title.value = note.title
                    _content.value = note.content
                    _color.value = note.color
                }
            }
        }
    }

    fun onTitleChange(title: String) { _title.value = title }
    fun onContentChange(content: String) { _content.value = content }
    fun onColorChange(color: Int) { _color.value = color }

    fun saveNote(onSaved: () -> Unit) {
        if (_title.value.isBlank() && _content.value.isBlank()) return

        viewModelScope.launch {
            val note = Note(
                id = if (_isNew) 0 else noteId,
                title = _title.value.ifBlank { "ไม่มีหัวข้อ" },
                content = _content.value,
                color = _color.value
            )
            if (_isNew) repository.insertNote(note)
            else repository.updateNote(note)
            onSaved()
        }
    }
}

@Composable
fun NoteEditScreen(
    viewModel: NoteEditViewModel = hiltViewModel(),
    onBack: () -> Unit
) {
    val title by viewModel.title.collectAsStateWithLifecycle()
    val content by viewModel.content.collectAsStateWithLifecycle()
    val hasChanges by viewModel.hasChanges.collectAsStateWithLifecycle()
    val scope = rememberCoroutineScope()

    BackHandler(enabled = hasChanges) {
        scope.launch {
            viewModel.saveNote(onSaved = onBack)
        }
    }

    Scaffold(
        topBar = {
            TopAppBar(
                title = { Text(if (title.isEmpty()) "Note ใหม่" else title) },
                navigationIcon = {
                    IconButton(onClick = {
                        viewModel.saveNote(onSaved = onBack)
                    }) {
                        Icon(Icons.Default.ArrowBack, "กลับ")
                    }
                }
            )
        }
    ) { padding ->
        Column(modifier = Modifier.padding(padding).padding(16.dp)) {
            OutlinedTextField(
                value = title,
                onValueChange = viewModel::onTitleChange,
                placeholder = { Text("หัวข้อ") },
                modifier = Modifier.fillMaxWidth(),
                textStyle = MaterialTheme.typography.headlineSmall,
                colors = TextFieldDefaults.colors(
                    focusedContainerColor = Color.Transparent,
                    unfocusedContainerColor = Color.Transparent,
                    focusedIndicatorColor = Color.Transparent,
                    unfocusedIndicatorColor = Color.Transparent
                )
            )
            OutlinedTextField(
                value = content,
                onValueChange = viewModel::onContentChange,
                placeholder = { Text("เขียนบันทึก...") },
                modifier = Modifier.fillMaxSize(),
                colors = TextFieldDefaults.colors(
                    focusedContainerColor = Color.Transparent,
                    unfocusedContainerColor = Color.Transparent,
                    focusedIndicatorColor = Color.Transparent,
                    unfocusedIndicatorColor = Color.Transparent
                )
            )
        }
    }
}
```

---

## ขั้นตอนที่ 829: Room กับ Relation

```kotlin
// One-to-Many: User มีหลาย Note
@Entity(tableName = "users")
data class User(
    @PrimaryKey val userId: Int,
    val name: String,
    val email: String
)

@Entity(
    tableName = "user_notes",
    foreignKeys = [ForeignKey(
        entity = User::class,
        parentColumns = ["userId"],
        childColumns = ["authorId"],
        onDelete = ForeignKey.CASCADE
    )]
)
data class UserNote(
    @PrimaryKey(autoGenerate = true) val noteId: Int = 0,
    val authorId: Int,
    val title: String,
    val content: String
)

// Data class สำหรับ relation
data class UserWithNotes(
    @Embedded val user: User,
    @Relation(
        parentColumn = "userId",
        entityColumn = "authorId"
    )
    val notes: List<UserNote>
)

// Dao สำหรับ relation
@Dao
interface UserDao {
    @Transaction
    @Query("SELECT * FROM users WHERE userId = :userId")
    fun getUserWithNotes(userId: Int): Flow<UserWithNotes?>

    @Transaction
    @Query("SELECT * FROM users")
    fun getAllUsersWithNotes(): Flow<List<UserWithNotes>>
}

// Many-to-Many: Note มีหลาย Tag, Tag มีหลาย Note
@Entity(tableName = "tags")
data class Tag(
    @PrimaryKey val tagId: Int,
    val name: String
)

@Entity(
    tableName = "note_tag_cross_ref",
    primaryKeys = ["noteId", "tagId"]
)
data class NoteTagCrossRef(
    val noteId: Int,
    val tagId: Int
)

data class NoteWithTags(
    @Embedded val note: Note,
    @Relation(
        parentColumn = "id",
        entityColumn = "tagId",
        associateBy = Junction(NoteTagCrossRef::class)
    )
    val tags: List<Tag>
)
```

---

## ขั้นตอนที่ 830: Testing Room Database

```kotlin
// androidTest/
import androidx.room.Room
import androidx.test.core.app.ApplicationProvider
import androidx.test.ext.junit.runners.AndroidJUnit4
import kotlinx.coroutines.flow.first
import kotlinx.coroutines.test.runTest
import org.junit.*
import org.junit.runner.RunWith

@RunWith(AndroidJUnit4::class)
class NoteDaoTest {
    private lateinit var database: AppDatabase
    private lateinit var noteDao: NoteDao

    @Before
    fun setup() {
        // ใช้ in-memory database สำหรับ test
        database = Room.inMemoryDatabaseBuilder(
            ApplicationProvider.getApplicationContext(),
            AppDatabase::class.java
        ).build()
        noteDao = database.noteDao()
    }

    @After
    fun teardown() {
        database.close()
    }

    @Test
    fun insertAndRetrieveNote() = runTest {
        val note = Note(title = "Test", content = "Content")
        val id = noteDao.insert(note)

        val retrieved = noteDao.getNoteById(id.toInt()).first()
        Assert.assertNotNull(retrieved)
        Assert.assertEquals("Test", retrieved?.title)
    }

    @Test
    fun searchNotes_returnsMatchingNotes() = runTest {
        noteDao.insert(Note(title = "Kotlin Tips", content = "..."))
        noteDao.insert(Note(title = "Android Guide", content = "Use Kotlin"))
        noteDao.insert(Note(title = "Python Tricks", content = "..."))

        val results = noteDao.searchNotes("kotlin").first()
        Assert.assertEquals(2, results.size)
    }

    @Test
    fun deleteNote_removesFromDatabase() = runTest {
        val note = Note(title = "Delete me", content = "")
        val id = noteDao.insert(note)
        val inserted = noteDao.getNoteById(id.toInt()).first()!!

        noteDao.delete(inserted)
        val deleted = noteDao.getNoteById(id.toInt()).first()
        Assert.assertNull(deleted)
    }
}
```

---

*Part 42 จบแล้ว | ก่อนหน้า: [Part 41](../part41/README.md) | ถัดไป: [Part 43](../part43/README.md)*
