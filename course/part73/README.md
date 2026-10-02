# Part 73: App Widgets & Glance
## ขั้นตอนที่ 1301-1325

---

## ขั้นตอนที่ 1301: App Widget Basics

```
App Widgets คืออะไร:
- แสดงข้อมูลบน Home Screen โดยไม่ต้องเปิด app
- ใช้ RemoteViews (ไม่ใช่ Compose ปกติ)
- Glance API - เขียน Widget ด้วย Compose-like syntax

ประเภท Widget:
1. Information Widget - แสดงข้อมูล (weather, clock)
2. Control Widget - ควบคุม app (music playback)
3. Hybrid Widget - ทั้งข้อมูลและควบคุม
4. Collection Widget - แสดงรายการ (ListView, GridView)
```

---

## ขั้นตอนที่ 1302: Glance Widget Setup

```kotlin
// build.gradle.kts
// implementation("androidx.glance:glance-appwidget:1.x")
// implementation("androidx.glance:glance-material3:1.x")

// 1. Define widget info
// res/xml/weather_widget_info.xml
/*
<appwidget-provider xmlns:android="http://schemas.android.com/apk/res/android"
    android:minWidth="110dp"
    android:minHeight="40dp"
    android:resizeMode="horizontal|vertical"
    android:updatePeriodMillis="1800000"
    android:previewLayout="@layout/weather_widget_preview"
    android:widgetCategory="home_screen"
    android:description="@string/widget_description"/>
*/

// 2. Glance Widget Receiver
class WeatherWidgetReceiver : GlanceAppWidgetReceiver() {
    override val glanceAppWidget: GlanceAppWidget = WeatherWidget()
}

// 3. Register in AndroidManifest.xml
/*
<receiver
    android:name=".WeatherWidgetReceiver"
    android:exported="true">
    <intent-filter>
        <action android:name="android.appwidget.action.APPWIDGET_UPDATE"/>
    </intent-filter>
    <meta-data
        android:name="android.appwidget.provider"
        android:resource="@xml/weather_widget_info"/>
</receiver>
*/
```

---

## ขั้นตอนที่ 1303: Glance Widget UI

```kotlin
class WeatherWidget : GlanceAppWidget() {
    
    override suspend fun provideGlance(context: Context, id: GlanceId) {
        // Fetch data before composing
        val weather = WeatherRepository(context).getCurrentWeather()
        
        provideContent {
            WeatherWidgetContent(weather)
        }
    }
    
    @Composable
    private fun WeatherWidgetContent(weather: WeatherData?) {
        GlanceTheme {
            Box(
                modifier = GlanceModifier
                    .fillMaxSize()
                    .background(GlanceTheme.colors.surface)
                    .padding(16.dp)
                    .cornerRadius(16.dp)
                    .clickable(actionStartActivity<MainActivity>())
            ) {
                if (weather == null) {
                    CircularProgressIndicator()
                } else {
                    Column {
                        // Temperature
                        Text(
                            text = "${weather.tempCelsius}°C",
                            style = TextStyle(
                                color = GlanceTheme.colors.onSurface,
                                fontSize = 32.sp,
                                fontWeight = FontWeight.Bold
                            )
                        )
                        
                        Spacer(modifier = GlanceModifier.height(4.dp))
                        
                        // Description
                        Text(
                            text = weather.description,
                            style = TextStyle(
                                color = GlanceTheme.colors.onSurfaceVariant,
                                fontSize = 14.sp
                            )
                        )
                        
                        Spacer(modifier = GlanceModifier.height(8.dp))
                        
                        // Location
                        Row(verticalAlignment = Alignment.CenterVertically) {
                            Image(
                                provider = ImageProvider(R.drawable.ic_location),
                                contentDescription = null,
                                modifier = GlanceModifier.size(12.dp)
                            )
                            Spacer(modifier = GlanceModifier.width(4.dp))
                            Text(
                                text = weather.location,
                                style = TextStyle(fontSize = 12.sp)
                            )
                        }
                    }
                }
            }
        }
    }
}

data class WeatherData(
    val tempCelsius: Int,
    val description: String,
    val location: String,
    val iconCode: String
)
```

---

## ขั้นตอนที่ 1304: Widget with Actions

```kotlin
// ============================================
// Quick Note Widget with Button Actions
// ============================================

class QuickNoteWidget : GlanceAppWidget() {
    
    override suspend fun provideGlance(context: Context, id: GlanceId) {
        val notes = NoteRepository(context).getRecentNotes(limit = 3)
        
        provideContent {
            QuickNoteContent(notes)
        }
    }
    
    @Composable
    private fun QuickNoteContent(notes: List<Note>) {
        Column(
            modifier = GlanceModifier
                .fillMaxSize()
                .background(GlanceTheme.colors.background)
                .padding(8.dp)
        ) {
            // Header
            Row(
                modifier = GlanceModifier.fillMaxWidth(),
                horizontalAlignment = Alignment.CenterHorizontally
            ) {
                Text(
                    "โน้ตล่าสุด",
                    style = TextStyle(fontWeight = FontWeight.Bold)
                )
                Spacer(GlanceModifier.defaultWeight())
                
                // Add note button
                Image(
                    provider = ImageProvider(R.drawable.ic_add),
                    contentDescription = "เพิ่มโน้ต",
                    modifier = GlanceModifier
                        .size(24.dp)
                        .clickable(
                            actionStartActivity(
                                Intent(LocalContext.current, MainActivity::class.java).apply {
                                    action = "ACTION_NEW_NOTE"
                                }
                            )
                        )
                )
            }
            
            Spacer(GlanceModifier.height(8.dp))
            
            // Note list
            if (notes.isEmpty()) {
                Text(
                    "ยังไม่มีโน้ต",
                    style = TextStyle(color = GlanceTheme.colors.onSurfaceVariant)
                )
            } else {
                notes.forEach { note ->
                    NoteItem(note)
                    Spacer(GlanceModifier.height(4.dp))
                }
            }
        }
    }
    
    @Composable
    private fun NoteItem(note: Note) {
        Box(
            modifier = GlanceModifier
                .fillMaxWidth()
                .background(GlanceTheme.colors.surfaceVariant)
                .padding(horizontal = 8.dp, vertical = 4.dp)
                .cornerRadius(8.dp)
                .clickable(
                    actionStartActivity(
                        Intent(LocalContext.current, MainActivity::class.java).apply {
                            action = "ACTION_OPEN_NOTE"
                            putExtra("note_id", note.id)
                        }
                    )
                )
        ) {
            Text(
                text = note.title,
                style = TextStyle(fontSize = 13.sp),
                maxLines = 1
            )
        }
    }
}

// Update widget from app
suspend fun updateWeatherWidget(context: Context) {
    val manager = GlanceAppWidgetManager(context)
    val glanceIds = manager.getGlanceIds(WeatherWidget::class.java)
    
    glanceIds.forEach { id ->
        WeatherWidget().update(context, id)
    }
}
```

---

## ขั้นตอนที่ 1305: Widget State with DataStore

```kotlin
// Widget ใช้ DataStore เพื่อ persist state
val widgetDataStore = context.createDataStore("widget_data")

// Save widget preferences
suspend fun saveWidgetPrefs(context: Context, widgetId: Int, location: String) {
    context.dataStore.edit { prefs ->
        prefs[stringPreferencesKey("widget_${widgetId}_location")] = location
    }
}

// Read in Widget
class ConfigurableWeatherWidget : GlanceAppWidget() {
    
    override suspend fun provideGlance(context: Context, id: GlanceId) {
        val widgetId = GlanceAppWidgetManager(context).getAppWidgetId(id)
        
        // Read stored location preference
        val location = context.dataStore.data
            .map { prefs ->
                prefs[stringPreferencesKey("widget_${widgetId}_location")] ?: "Bangkok"
            }
            .first()
        
        val weather = WeatherRepository(context).getWeatherForLocation(location)
        
        provideContent {
            WeatherContent(weather, location)
        }
    }
}

// Widget Configuration Activity
@AndroidEntryPoint
class WidgetConfigActivity : ComponentActivity() {
    
    private val widgetId by lazy {
        intent?.extras?.getInt(
            AppWidgetManager.EXTRA_APPWIDGET_ID,
            AppWidgetManager.INVALID_APPWIDGET_ID
        ) ?: AppWidgetManager.INVALID_APPWIDGET_ID
    }
    
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        
        // Set result to CANCELED in case user backs out
        setResult(RESULT_CANCELED)
        
        setContent {
            WidgetConfigScreen(
                onConfirm = { location ->
                    lifecycleScope.launch {
                        saveWidgetPrefs(this@WidgetConfigActivity, widgetId, location)
                        
                        // Trigger widget update
                        updateWeatherWidget(this@WidgetConfigActivity)
                        
                        val result = Intent().apply {
                            putExtra(AppWidgetManager.EXTRA_APPWIDGET_ID, widgetId)
                        }
                        setResult(RESULT_OK, result)
                        finish()
                    }
                }
            )
        }
    }
}
```

---

## แบบฝึกหัด Part 73

```kotlin
// แบบฝึกหัด: Habit Tracker Widget

// สร้าง HabitTrackerWidget ที่:
// 1. แสดงรายการ habits ประจำวัน (สูงสุด 5 รายการ)
// 2. มี checkbox toggle สำหรับแต่ละ habit
// 3. แสดง progress bar รวม (X/5 เสร็จแล้ว)
// 4. กด habit → เปิด app ที่ HabitDetailScreen
// 5. State persist ด้วย DataStore

data class Habit(
    val id: Long,
    val name: String,
    val isCompleted: Boolean,
    val streak: Int
)

class HabitTrackerWidget : GlanceAppWidget() {
    override suspend fun provideGlance(context: Context, id: GlanceId) {
        // TODO: implement
    }
}
```

---

*Part 73 จบแล้ว | ก่อนหน้า: [Part 72](../part72/README.md) | ถัดไป: [Part 74](../part74/README.md)*
