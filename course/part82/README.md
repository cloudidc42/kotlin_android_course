# Part 82: Compose Multiplatform (Desktop & iOS)
## ขั้นตอนที่ 1526-1550

---

## ขั้นตอนที่ 1526: Compose Multiplatform Overview

```
Compose Multiplatform (by JetBrains):
- เขียน UI ด้วย Compose ใช้ได้กับทุก platform
- Android, iOS, Desktop (Windows/macOS/Linux), Web (experimental)

KMP + Compose Multiplatform Architecture:

shared/
├── commonMain/         (Kotlin/Compose logic)
│   ├── models/
│   ├── viewmodels/
│   ├── repositories/
│   └── ui/            (Compose screens)
├── androidMain/        (Android-specific)
│   └── actual implementations
├── iosMain/            (iOS-specific)
│   └── actual implementations
└── desktopMain/        (Desktop-specific)
    └── actual implementations

app-android/  (Android app entry point)
app-ios/      (Xcode project)
app-desktop/  (Desktop entry point)
```

---

## ขั้นตอนที่ 1527: Desktop App Entry Point

```kotlin
// desktopMain/kotlin/main.kt
import androidx.compose.ui.window.*

fun main() = application {
    Window(
        onCloseRequest = ::exitApplication,
        title = "My App",
        state = rememberWindowState(
            width = 1280.dp,
            height = 800.dp
        )
    ) {
        // Compose UI
        App()
    }
}

// Shared App composable
@Composable
fun App() {
    MaterialTheme {
        AppNavigation()
    }
}

// Desktop-specific features
@Composable
fun DesktopMenu() {
    MenuBar {
        Menu("File") {
            Item("New", onClick = { /* create new */ })
            Item("Open...", onClick = { /* open dialog */ })
            Separator()
            Item("Quit", onClick = ::exitApplication)
        }
        Menu("Edit") {
            Item("Copy", onClick = { /* copy */ })
            Item("Paste", onClick = { /* paste */ })
        }
    }
}
```

---

## ขั้นตอนที่ 1528: File Picker (Cross-Platform)

```kotlin
// expect/actual for file operations

// commonMain
expect class FileManager {
    suspend fun pickFile(fileTypes: List<String>): File?
    suspend fun saveFile(data: ByteArray, fileName: String): Boolean
}

// androidMain
actual class FileManager(private val context: Context) {
    
    private var filePicker: ActivityResultLauncher<Array<String>>? = null
    
    actual suspend fun pickFile(fileTypes: List<String>): File? {
        // Use ActivityResultLauncher for file picker
        return null // simplified
    }
    
    actual suspend fun saveFile(data: ByteArray, fileName: String): Boolean {
        return try {
            val file = File(context.filesDir, fileName)
            file.writeBytes(data)
            true
        } catch (e: Exception) { false }
    }
}

// desktopMain
actual class FileManager {
    
    actual suspend fun pickFile(fileTypes: List<String>): File? {
        return withContext(Dispatchers.IO) {
            val chooser = JFileChooser()
            
            val filter = FileNameExtensionFilter(
                "Files (${fileTypes.joinToString(", ")})",
                *fileTypes.map { it.removePrefix(".") }.toTypedArray()
            )
            chooser.fileFilter = filter
            
            val result = chooser.showOpenDialog(null)
            if (result == JFileChooser.APPROVE_OPTION) {
                chooser.selectedFile
            } else {
                null
            }
        }
    }
    
    actual suspend fun saveFile(data: ByteArray, fileName: String): Boolean {
        return withContext(Dispatchers.IO) {
            try {
                File(fileName).writeBytes(data)
                true
            } catch (e: Exception) { false }
        }
    }
}

// iosMain  
actual class FileManager {
    actual suspend fun pickFile(fileTypes: List<String>): File? {
        // UIDocumentPickerViewController
        return null // iOS implementation
    }
    
    actual suspend fun saveFile(data: ByteArray, fileName: String): Boolean {
        return try {
            val documentsDir = NSSearchPathForDirectoriesInDomains(
                NSDocumentDirectory, NSUserDomainMask, true
            ).first() as String
            val filePath = "$documentsDir/$fileName"
            (data.toNSData()).writeToFile(filePath, atomically = true)
        } catch (e: Exception) { false }
    }
}
```

---

## ขั้นตอนที่ 1529: Desktop-specific UI Components

```kotlin
// Desktop context menu
@Composable
fun DesktopContextMenu(
    content: @Composable () -> Unit
) {
    ContextMenuArea(
        items = {
            listOf(
                ContextMenuItem("Cut") { /* cut */ },
                ContextMenuItem("Copy") { /* copy */ },
                ContextMenuItem("Paste") { /* paste */ }
            )
        }
    ) {
        content()
    }
}

// Desktop drag-and-drop
@Composable
fun DropTarget(
    onFilesDropped: (List<File>) -> Unit
) {
    var isDragging by remember { mutableStateOf(false) }
    
    Box(
        modifier = Modifier
            .fillMaxSize()
            .background(if (isDragging) Color.Blue.copy(alpha = 0.1f) else Color.Transparent)
            .onExternalDrag(
                onDragStart = { isDragging = true },
                onDragExit = { isDragging = false },
                onDrop = { state ->
                    isDragging = false
                    val files = state.dragData.transferData(DataFlavor.javaFileListFlavor)
                        as? List<File> ?: return@onExternalDrag
                    onFilesDropped(files)
                }
            )
    ) {
        if (isDragging) {
            Text(
                "วางไฟล์ที่นี่",
                modifier = Modifier.align(Alignment.Center),
                style = MaterialTheme.typography.headlineMedium
            )
        }
    }
}

// Keyboard shortcuts
@Composable
fun KeyboardShortcuts() {
    val keyEventHandler: (KeyEvent) -> Boolean = { keyEvent ->
        when {
            keyEvent.isMetaPressed && keyEvent.key == Key.S -> {
                // Cmd/Ctrl+S: Save
                true
            }
            keyEvent.isMetaPressed && keyEvent.key == Key.Z -> {
                // Cmd/Ctrl+Z: Undo
                true
            }
            else -> false
        }
    }
    
    // Apply to focused component
}
```

---

## ขั้นตอนที่ 1530: System Tray & Notifications (Desktop)

```kotlin
// System Tray icon
fun main() = application {
    val trayState = rememberTrayState()
    
    Tray(
        state = trayState,
        icon = painterResource("icon.png"),
        menu = {
            Item("Open", onClick = { /* show window */ })
            Separator()
            Item("Quit", onClick = ::exitApplication)
        }
    )
    
    // Show notification from tray
    LaunchedEffect(Unit) {
        delay(5000)
        trayState.sendNotification(
            Notification(
                title = "My App",
                message = "การซิงค์ข้อมูลเสร็จสิ้น",
                type = Notification.Type.Info
            )
        )
    }
    
    Window(
        onCloseRequest = ::exitApplication,
        title = "My Desktop App"
    ) {
        App()
    }
}

// Multi-window support
@Composable
fun MultiWindowApp() {
    var showSettings by remember { mutableStateOf(false) }
    
    if (showSettings) {
        Window(
            onCloseRequest = { showSettings = false },
            title = "Settings",
            state = rememberWindowState(width = 600.dp, height = 400.dp)
        ) {
            SettingsScreen()
        }
    }
    
    // Main window content
    MainContent(onOpenSettings = { showSettings = true })
}
```

---

## แบบฝึกหัด Part 82

```kotlin
// แบบฝึกหัด: Cross-Platform Note App

// สร้าง Note App ที่ทำงานบน Android และ Desktop
// Shared code:
// - Note data model
// - NoteRepository (with expect/actual for storage)
// - NoteViewModel
// - NoteListScreen, NoteDetailScreen

// commonMain
expect class NoteStorage {
    suspend fun loadNotes(): List<Note>
    suspend fun saveNote(note: Note)
    suspend fun deleteNote(id: Long)
}

// androidMain - ใช้ Room
actual class NoteStorage(private val dao: NoteDao) {
    actual suspend fun loadNotes() = dao.getAll()
    actual suspend fun saveNote(note: Note) = dao.insert(note.toEntity())
    actual suspend fun deleteNote(id: Long) = dao.deleteById(id)
}

// desktopMain - ใช้ SQLite (via sqlite-jdbc)
actual class NoteStorage(private val db: Connection) {
    actual suspend fun loadNotes(): List<Note> {
        return withContext(Dispatchers.IO) {
            // TODO: implement with JDBC
            emptyList()
        }
    }
    actual suspend fun saveNote(note: Note) { TODO() }
    actual suspend fun deleteNote(id: Long) { TODO() }
}
```

---

*Part 82 จบแล้ว | ก่อนหน้า: [Part 81](../part81/README.md) | ถัดไป: [Part 83](../part83/README.md)*
