# Part 66: Background Processing & Services
## ขั้นตอนที่ 1126-1150

---

## ขั้นตอนที่ 1126: Background Processing Options

```
Android Background Processing:

├── Immediate Work
│   ├── Coroutines (ใน-app)
│   └── Threads (ใน-app)
│
├── Long-Running Work
│   ├── Foreground Service (user-visible)
│   └── WorkManager (system-managed)
│
└── Deferred Work
    ├── WorkManager (recommended)
    ├── AlarmManager (exact timing)
    └── JobScheduler (API 21+)

ใช้งานเมื่อ:
- WorkManager: sync, upload, background processing
- Foreground Service: music, navigation, download
- AlarmManager: alarm clock, calendar reminder
```

---

## ขั้นตอนที่ 1127: Advanced WorkManager

```kotlin
// ============================================
// Complex Work Chains
// ============================================

class SyncManager @Inject constructor(
    private val workManager: WorkManager
) {
    
    fun scheduleDailySync() {
        val constraints = Constraints.Builder()
            .setRequiredNetworkType(NetworkType.CONNECTED)
            .setRequiresBatteryNotLow(true)
            .build()
        
        val syncWork = PeriodicWorkRequestBuilder<SyncWorker>(
            repeatInterval = 24,
            repeatIntervalTimeUnit = TimeUnit.HOURS,
            flexTimeInterval = 2,
            flexTimeIntervalUnit = TimeUnit.HOURS
        )
            .setConstraints(constraints)
            .setBackoffCriteria(BackoffPolicy.EXPONENTIAL, 30, TimeUnit.MINUTES)
            .build()
        
        workManager.enqueueUniquePeriodicWork(
            "daily_sync",
            ExistingPeriodicWorkPolicy.KEEP,
            syncWork
        )
    }
    
    fun scheduleImageUpload(imageUris: List<Uri>) {
        // สร้าง work chain
        val compressWork = OneTimeWorkRequestBuilder<ImageCompressWorker>()
            .setInputData(
                Data.Builder()
                    .putStringArray("image_uris", imageUris.map { it.toString() }.toTypedArray())
                    .build()
            )
            .build()
        
        val uploadWork = OneTimeWorkRequestBuilder<ImageUploadWorker>()
            .setConstraints(
                Constraints.Builder()
                    .setRequiredNetworkType(NetworkType.CONNECTED)
                    .build()
            )
            .build()
        
        val notifyWork = OneTimeWorkRequestBuilder<NotifyUploadCompleteWorker>().build()
        
        // chain: compress → upload → notify
        workManager
            .beginWith(compressWork)
            .then(uploadWork)
            .then(notifyWork)
            .enqueue()
    }
    
    fun observeSyncStatus(): LiveData<List<WorkInfo>> {
        return workManager.getWorkInfosForUniqueWorkLiveData("daily_sync")
    }
}

// ============================================
// SyncWorker กับ Hilt
// ============================================

@HiltWorker
class SyncWorker @AssistedInject constructor(
    @Assisted context: Context,
    @Assisted params: WorkerParameters,
    private val userRepository: UserRepository,
    private val productRepository: ProductRepository,
    private val orderRepository: OrderRepository
) : CoroutineWorker(context, params) {
    
    override suspend fun doWork(): Result {
        return try {
            setForeground(createForegroundInfo())
            
            // Sync ทีละ step
            setProgress(createProgress("Syncing users...", 0))
            userRepository.sync()
            
            setProgress(createProgress("Syncing products...", 33))
            productRepository.sync()
            
            setProgress(createProgress("Syncing orders...", 66))
            orderRepository.sync()
            
            setProgress(createProgress("Complete", 100))
            
            Result.success(
                Data.Builder()
                    .putLong("sync_time", System.currentTimeMillis())
                    .build()
            )
            
        } catch (e: Exception) {
            if (runAttemptCount < 3) {
                Result.retry()
            } else {
                Result.failure(
                    Data.Builder()
                        .putString("error", e.message)
                        .build()
                )
            }
        }
    }
    
    private fun createForegroundInfo(): ForegroundInfo {
        val notification = NotificationCompat.Builder(applicationContext, CHANNEL_ID)
            .setContentTitle("กำลัง Sync ข้อมูล")
            .setSmallIcon(R.drawable.ic_sync)
            .setOngoing(true)
            .build()
        
        return ForegroundInfo(NOTIFICATION_ID, notification)
    }
    
    private fun createProgress(message: String, percent: Int): Data {
        return Data.Builder()
            .putString("message", message)
            .putInt("percent", percent)
            .build()
    }
    
    companion object {
        const val CHANNEL_ID = "sync_channel"
        const val NOTIFICATION_ID = 1001
    }
}
```

---

## ขั้นตอนที่ 1128: Foreground Services

```kotlin
// ============================================
// Foreground Service สำหรับ Music Player
// ============================================

class MusicService : Service() {
    
    private val binder = MusicBinder()
    private var mediaPlayer: MediaPlayer? = null
    private lateinit var notificationManager: NotificationManagerCompat
    
    inner class MusicBinder : Binder() {
        fun getService(): MusicService = this@MusicService
    }
    
    override fun onBind(intent: Intent?): IBinder = binder
    
    override fun onCreate() {
        super.onCreate()
        notificationManager = NotificationManagerCompat.from(this)
        createNotificationChannel()
    }
    
    fun play(track: Track) {
        mediaPlayer?.release()
        mediaPlayer = MediaPlayer.create(this, track.uri)
        mediaPlayer?.start()
        
        startForeground(NOTIFICATION_ID, buildNotification(track, isPlaying = true))
    }
    
    fun pause() {
        mediaPlayer?.pause()
        // Update notification
    }
    
    fun stop() {
        mediaPlayer?.stop()
        mediaPlayer?.release()
        mediaPlayer = null
        stopForeground(STOP_FOREGROUND_REMOVE)
        stopSelf()
    }
    
    private fun buildNotification(track: Track, isPlaying: Boolean): Notification {
        val playPauseAction = if (isPlaying) {
            NotificationCompat.Action(
                R.drawable.ic_pause, "Pause",
                PendingIntent.getService(this, 0,
                    Intent(this, MusicService::class.java).setAction("PAUSE"),
                    PendingIntent.FLAG_IMMUTABLE)
            )
        } else {
            NotificationCompat.Action(
                R.drawable.ic_play, "Play",
                PendingIntent.getService(this, 0,
                    Intent(this, MusicService::class.java).setAction("PLAY"),
                    PendingIntent.FLAG_IMMUTABLE)
            )
        }
        
        return NotificationCompat.Builder(this, CHANNEL_ID)
            .setContentTitle(track.title)
            .setContentText(track.artist)
            .setSmallIcon(R.drawable.ic_music)
            .addAction(playPauseAction)
            .setStyle(
                androidx.media.app.NotificationCompat.MediaStyle()
                    .setShowActionsInCompactView(0)
            )
            .build()
    }
    
    private fun createNotificationChannel() {
        val channel = NotificationChannel(
            CHANNEL_ID,
            "Music Playback",
            NotificationManager.IMPORTANCE_LOW
        ).apply {
            description = "Music player controls"
        }
        
        val manager = getSystemService(NotificationManager::class.java)
        manager.createNotificationChannel(channel)
    }
    
    override fun onDestroy() {
        mediaPlayer?.release()
        super.onDestroy()
    }
    
    companion object {
        const val CHANNEL_ID = "music_channel"
        const val NOTIFICATION_ID = 100
    }
}
```

---

## ขั้นตอนที่ 1129: BroadcastReceiver

```kotlin
// ============================================
// BroadcastReceiver สำหรับ System Events
// ============================================

// รับ network change
class NetworkChangeReceiver : BroadcastReceiver() {
    
    override fun onReceive(context: Context, intent: Intent) {
        val connectivityManager = context.getSystemService(ConnectivityManager::class.java)
        val isConnected = connectivityManager.activeNetwork != null
        
        if (isConnected) {
            // Trigger sync
            WorkManager.getInstance(context)
                .enqueueUniqueWork(
                    "post_network_sync",
                    ExistingWorkPolicy.REPLACE,
                    OneTimeWorkRequestBuilder<SyncWorker>().build()
                )
        }
    }
}

// Register ใน AndroidManifest (static):
// <receiver android:name=".NetworkChangeReceiver"
//           android:exported="false">
//     <intent-filter>
//         <action android:name="android.net.conn.CONNECTIVITY_CHANGE"/>
//     </intent-filter>
// </receiver>

// Register dynamically (better):
class MainActivity : ComponentActivity() {
    
    private val receiver = NetworkChangeReceiver()
    
    override fun onStart() {
        super.onStart()
        val filter = IntentFilter(ConnectivityManager.CONNECTIVITY_ACTION)
        registerReceiver(receiver, filter)
    }
    
    override fun onStop() {
        super.onStop()
        unregisterReceiver(receiver)
    }
}

// ============================================
// LocalBroadcastManager (ภายใน app เท่านั้น)
// ============================================

// ส่ง
LocalBroadcastManager.getInstance(context)
    .sendBroadcast(Intent("com.example.DATA_UPDATED").apply {
        putExtra("type", "products")
    })

// รับ
val localReceiver = object : BroadcastReceiver() {
    override fun onReceive(context: Context, intent: Intent) {
        val type = intent.getStringExtra("type")
        viewModel.refreshData(type)
    }
}

LocalBroadcastManager.getInstance(context)
    .registerReceiver(localReceiver, IntentFilter("com.example.DATA_UPDATED"))
```

---

## แบบฝึกหัด Part 66

```kotlin
// แบบฝึกหัด: File Sync Service

// TODO: สร้าง FileSyncService ที่:
// 1. ทำงานเป็น Foreground Service
// 2. Upload files จาก local storage ไปยัง server
// 3. แสดง progress ใน notification
// 4. รองรับ pause/resume/cancel
// 5. Retry ถ้า upload ล้มเหลว
// 6. บันทึก upload status ใน Room

@HiltWorker
class FileUploadWorker @AssistedInject constructor(
    @Assisted context: Context,
    @Assisted params: WorkerParameters,
    private val fileRepository: FileRepository,
    private val uploadService: UploadService
) : CoroutineWorker(context, params) {
    
    override suspend fun doWork(): Result {
        val fileId = inputData.getString("file_id") ?: return Result.failure()
        
        return try {
            val file = fileRepository.getFile(fileId)
                ?: return Result.failure()
            
            // TODO: implement upload with progress
            // - setForeground() สำหรับ notification
            // - emit progress ผ่าน setProgress()
            // - handle cancellation
            
            Result.success()
        } catch (e: CancellationException) {
            throw e  // always re-throw cancellation
        } catch (e: Exception) {
            if (runAttemptCount < 3) Result.retry()
            else Result.failure()
        }
    }
}
```

---

*Part 66 จบแล้ว | ก่อนหน้า: [Part 65](../part65/README.md) | ถัดไป: [Part 67](../part67/README.md)*
