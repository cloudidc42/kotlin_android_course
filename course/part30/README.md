# Part 30: WorkManager และ Background Tasks
## ขั้นตอนที่ 726-750

---

## ขั้นตอนที่ 726: WorkManager คืออะไร?

WorkManager ทำงาน background tasks ที่ต้องรันแน่นอน แม้ app ปิด หรือ device reboot

```
Background Work Options:
                    
  ┌────────────┬─────────────────────────────────┐
  │  Duration  │         When to use             │
  ├────────────┼─────────────────────────────────┤
  │ Short      │ Coroutines, Thread              │
  │ (<10 min)  │ foreground operation            │
  ├────────────┼─────────────────────────────────┤
  │ Long       │ Foreground Service               │
  │ (immediate)│ (เช่น download, upload)         │
  ├────────────┼─────────────────────────────────┤
  │ Deferrable │ WorkManager ← ใช้นี่!           │
  │ + Reliable │ (sync, backup, cleanup)         │
  └────────────┴─────────────────────────────────┘
```

---

## ขั้นตอนที่ 727: Worker

```kotlin
// build.gradle.kts
// implementation("androidx.work:work-runtime-ktx:2.9.x")
// implementation("androidx.hilt:hilt-work:1.2.x")
// ksp("androidx.hilt:hilt-compiler:1.2.x")

// ============================================
// Basic Worker
// ============================================

class SyncWorker(
    context: Context,
    params: WorkerParameters
) : CoroutineWorker(context, params) {
    
    override suspend fun doWork(): Result {
        return try {
            // รับ input data
            val userId = inputData.getLong("USER_ID", -1)
            
            // ทำ work
            performSync(userId)
            
            // ส่ง output data
            val output = workDataOf("SYNCED_COUNT" to 100)
            Result.success(output)
            
        } catch (e: Exception) {
            if (runAttemptCount < 3) {
                Result.retry()  // ลองใหม่
            } else {
                Result.failure(workDataOf("ERROR" to e.message))
            }
        }
    }
    
    private suspend fun performSync(userId: Long) {
        // Actual sync logic
        delay(2000)
        println("Synced user $userId")
    }
}

// Worker ที่แสดง Progress
class DownloadWorker(
    context: Context,
    params: WorkerParameters
) : CoroutineWorker(context, params) {
    
    override suspend fun doWork(): Result {
        val url = inputData.getString("URL") ?: return Result.failure()
        val fileName = inputData.getString("FILE_NAME") ?: return Result.failure()
        
        try {
            downloadFile(url, fileName) { progress ->
                setProgress(workDataOf("PROGRESS" to progress))
            }
            return Result.success()
        } catch (e: Exception) {
            return Result.failure(workDataOf("ERROR" to e.message))
        }
    }
    
    private suspend fun downloadFile(
        url: String,
        fileName: String,
        onProgress: suspend (Int) -> Unit
    ) {
        for (i in 0..100 step 10) {
            delay(100)
            onProgress(i)
        }
    }
}
```

---

## ขั้นตอนที่ 728: Scheduling Work

```kotlin
class WorkScheduler(private val context: Context) {
    
    private val workManager = WorkManager.getInstance(context)
    
    // ============================================
    // One-time work
    // ============================================
    
    fun scheduleOneTimeSync(userId: Long) {
        val inputData = workDataOf("USER_ID" to userId)
        
        val constraints = Constraints.Builder()
            .setRequiredNetworkType(NetworkType.CONNECTED)
            .setRequiresBatteryNotLow(true)
            .build()
        
        val syncRequest = OneTimeWorkRequestBuilder<SyncWorker>()
            .setInputData(inputData)
            .setConstraints(constraints)
            .setBackoffCriteria(
                BackoffPolicy.EXPONENTIAL,
                WorkRequest.MIN_BACKOFF_MILLIS,
                TimeUnit.MILLISECONDS
            )
            .addTag("sync")
            .build()
        
        workManager.enqueue(syncRequest)
    }
    
    // ============================================
    // Periodic work
    // ============================================
    
    fun schedulePeriodicSync() {
        val constraints = Constraints.Builder()
            .setRequiredNetworkType(NetworkType.CONNECTED)
            .build()
        
        val periodicRequest = PeriodicWorkRequestBuilder<SyncWorker>(
            repeatInterval = 6,
            repeatIntervalTimeUnit = TimeUnit.HOURS,
            flexTimeInterval = 30,
            flexTimeIntervalUnit = TimeUnit.MINUTES
        )
        .setConstraints(constraints)
        .setInitialDelay(1, TimeUnit.HOURS)
        .addTag("periodic_sync")
        .build()
        
        // ใช้ enqueueUniquePeriodicWork เพื่อป้องกัน duplicate
        workManager.enqueueUniquePeriodicWork(
            "periodic_sync",
            ExistingPeriodicWorkPolicy.KEEP,
            periodicRequest
        )
    }
    
    fun cancelPeriodicSync() {
        workManager.cancelUniqueWork("periodic_sync")
    }
    
    // ============================================
    // Unique work
    // ============================================
    
    fun scheduleUniqueDownload(url: String, fileName: String) {
        val inputData = workDataOf(
            "URL" to url,
            "FILE_NAME" to fileName
        )
        
        val downloadRequest = OneTimeWorkRequestBuilder<DownloadWorker>()
            .setInputData(inputData)
            .build()
        
        workManager.enqueueUniqueWork(
            "download_$fileName",
            ExistingWorkPolicy.KEEP,  // ถ้ามีอยู่แล้ว ไม่ต้องทำใหม่
            downloadRequest
        )
    }
    
    // ============================================
    // Work chain
    // ============================================
    
    fun scheduleChainedWork() {
        val downloadWork = OneTimeWorkRequestBuilder<DownloadWorker>().build()
        val compressWork = OneTimeWorkRequestBuilder<CompressWorker>().build()
        val uploadWork = OneTimeWorkRequestBuilder<UploadWorker>().build()
        
        workManager
            .beginWith(downloadWork)
            .then(compressWork)
            .then(uploadWork)
            .enqueue()
    }
    
    // Parallel then sequential
    fun scheduleParallelWork() {
        val download1 = OneTimeWorkRequestBuilder<DownloadWorker>().build()
        val download2 = OneTimeWorkRequestBuilder<DownloadWorker>().build()
        val mergeWork = OneTimeWorkRequestBuilder<MergeWorker>().build()
        
        workManager
            .beginWith(listOf(download1, download2))  // parallel
            .then(mergeWork)  // sequential after both complete
            .enqueue()
    }
}
```

---

## ขั้นตอนที่ 729: Observing Work Status

```kotlin
@Composable
fun DownloadScreen(
    workManager: WorkManager = WorkManager.getInstance(LocalContext.current)
) {
    var downloadState by remember { mutableStateOf<WorkInfo.State?>(null) }
    var progress by remember { mutableIntStateOf(0) }
    var workId by remember { mutableStateOf<UUID?>(null) }
    
    // Observe work status
    workId?.let { id ->
        val workInfo by workManager
            .getWorkInfoByIdLiveData(id)
            .observeAsState()
        
        LaunchedEffect(workInfo) {
            workInfo?.let { info ->
                downloadState = info.state
                progress = info.progress.getInt("PROGRESS", 0)
                
                when (info.state) {
                    WorkInfo.State.SUCCEEDED -> {
                        println("Download complete!")
                    }
                    WorkInfo.State.FAILED -> {
                        val error = info.outputData.getString("ERROR")
                        println("Error: $error")
                    }
                    else -> {}
                }
            }
        }
    }
    
    Column(
        modifier = Modifier.fillMaxSize().padding(16.dp),
        horizontalAlignment = Alignment.CenterHorizontally
    ) {
        when (downloadState) {
            null -> Text("พร้อมดาวน์โหลด")
            WorkInfo.State.ENQUEUED -> Text("รอคิว...")
            WorkInfo.State.RUNNING -> {
                Text("กำลังดาวน์โหลด: $progress%")
                LinearProgressIndicator(progress = { progress / 100f })
            }
            WorkInfo.State.SUCCEEDED -> Text("ดาวน์โหลดเสร็จสิ้น!")
            WorkInfo.State.FAILED -> Text("ดาวน์โหลดล้มเหลว")
            WorkInfo.State.CANCELLED -> Text("ยกเลิกแล้ว")
            WorkInfo.State.BLOCKED -> Text("รอ dependencies...")
        }
        
        Spacer(Modifier.height(16.dp))
        
        Button(
            onClick = {
                val request = OneTimeWorkRequestBuilder<DownloadWorker>()
                    .setInputData(workDataOf(
                        "URL" to "https://example.com/file.zip",
                        "FILE_NAME" to "file.zip"
                    ))
                    .build()
                
                workId = request.id
                WorkManager.getInstance(LocalContext.current).enqueue(request)
            },
            enabled = downloadState == null || downloadState == WorkInfo.State.SUCCEEDED
        ) {
            Text("เริ่มดาวน์โหลด")
        }
        
        if (downloadState == WorkInfo.State.RUNNING) {
            Button(
                onClick = {
                    workId?.let { WorkManager.getInstance(LocalContext.current).cancelWorkById(it) }
                }
            ) {
                Text("ยกเลิก")
            }
        }
    }
}
```

---

## ขั้นตอนที่ 730: Hilt Worker

```kotlin
// ============================================
// Hilt Worker - inject dependencies ใน Worker
// ============================================

@HiltWorker
class SyncDataWorker @AssistedInject constructor(
    @Assisted context: Context,
    @Assisted params: WorkerParameters,
    private val userRepository: UserRepository,
    private val postRepository: PostRepository,
    @IoDispatcher private val ioDispatcher: CoroutineDispatcher
) : CoroutineWorker(context, params) {
    
    override suspend fun doWork(): Result = withContext(ioDispatcher) {
        try {
            setForeground(createForegroundInfo("กำลัง sync..."))
            
            // Sync users
            setProgress(workDataOf("STATUS" to "Syncing users"))
            userRepository.refreshUsers()
            
            // Sync posts
            setProgress(workDataOf("STATUS" to "Syncing posts"))
            postRepository.refreshPosts()
            
            Result.success(workDataOf("SYNCED_AT" to System.currentTimeMillis()))
        } catch (e: CancellationException) {
            throw e
        } catch (e: Exception) {
            if (runAttemptCount < 3) Result.retry()
            else Result.failure(workDataOf("ERROR" to e.message))
        }
    }
    
    private fun createForegroundInfo(progress: String): ForegroundInfo {
        val notification = NotificationCompat.Builder(applicationContext, CHANNEL_ID)
            .setContentTitle("กำลัง Sync ข้อมูล")
            .setContentText(progress)
            .setSmallIcon(R.drawable.ic_sync)
            .setOngoing(true)
            .build()
        
        return ForegroundInfo(NOTIFICATION_ID, notification)
    }
    
    companion object {
        const val CHANNEL_ID = "sync_channel"
        const val NOTIFICATION_ID = 1001
    }
}

// Setup HiltWorkerFactory ใน Application
@HiltAndroidApp
class MyApplication : Application(), Configuration.Provider {
    
    @Inject
    lateinit var workerFactory: HiltWorkerFactory
    
    override val workManagerConfiguration: Configuration
        get() = Configuration.Builder()
            .setWorkerFactory(workerFactory)
            .build()
}
```

---

## ขั้นตอนที่ 731: Data Sync Pattern

```kotlin
// ============================================
// Complete Data Sync Pattern
// ============================================

class DataSyncManager @Inject constructor(
    @ApplicationContext private val context: Context
) {
    private val workManager = WorkManager.getInstance(context)
    
    // Initial sync (on app first launch)
    fun initialSync(): UUID {
        val request = OneTimeWorkRequestBuilder<FullSyncWorker>()
            .setConstraints(
                Constraints.Builder()
                    .setRequiredNetworkType(NetworkType.CONNECTED)
                    .build()
            )
            .build()
        
        workManager.enqueueUniqueWork(
            "initial_sync",
            ExistingWorkPolicy.KEEP,
            request
        )
        
        return request.id
    }
    
    // Periodic background sync
    fun startPeriodicSync() {
        val request = PeriodicWorkRequestBuilder<BackgroundSyncWorker>(
            repeatInterval = 15,
            repeatIntervalTimeUnit = TimeUnit.MINUTES
        )
        .setConstraints(
            Constraints.Builder()
                .setRequiredNetworkType(NetworkType.CONNECTED)
                .build()
        )
        .build()
        
        workManager.enqueueUniquePeriodicWork(
            "periodic_sync",
            ExistingPeriodicWorkPolicy.UPDATE,
            request
        )
    }
    
    // On-demand sync
    fun syncNow() {
        val request = OneTimeWorkRequestBuilder<BackgroundSyncWorker>()
            .setExpedited(OutOfQuotaPolicy.RUN_AS_NON_EXPEDITED_WORK_REQUEST)
            .build()
        
        workManager.enqueue(request)
    }
    
    // Cancel all sync
    fun cancelAll() {
        workManager.cancelAllWorkByTag("sync")
    }
    
    // Observe sync status
    fun observeSyncStatus(): LiveData<List<WorkInfo>> {
        return workManager.getWorkInfosByTagLiveData("sync")
    }
}
```

---

## แบบฝึกหัด Part 30

```kotlin
// แบบฝึกหัด: Image Processing Worker

// สร้าง Worker ที่:
// 1. รับ image URI จาก input data
// 2. Resize image เป็น 3 ขนาด (thumbnail, medium, large)
// 3. รายงาน progress ในแต่ละขั้น
// 4. Return URIs ของ images ที่ resize แล้ว

class ImageProcessingWorker(
    context: Context,
    params: WorkerParameters
) : CoroutineWorker(context, params) {
    
    override suspend fun doWork(): Result {
        val imageUri = inputData.getString("IMAGE_URI") 
            ?: return Result.failure()
        
        return try {
            // Step 1: Load image (25%)
            setProgress(workDataOf("PROGRESS" to 0, "STEP" to "กำลังโหลดรูป"))
            val bitmap = loadImage(imageUri)
            setProgress(workDataOf("PROGRESS" to 25))
            
            // Step 2: Create thumbnail (50%)
            setProgress(workDataOf("PROGRESS" to 25, "STEP" to "สร้าง thumbnail"))
            val thumbnailUri = resizeAndSave(bitmap, 100, 100, "thumbnail_")
            setProgress(workDataOf("PROGRESS" to 50))
            
            // Step 3: Create medium (75%)
            setProgress(workDataOf("PROGRESS" to 50, "STEP" to "สร้าง medium"))
            val mediumUri = resizeAndSave(bitmap, 400, 400, "medium_")
            setProgress(workDataOf("PROGRESS" to 75))
            
            // Step 4: Create large (100%)
            setProgress(workDataOf("PROGRESS" to 75, "STEP" to "สร้าง large"))
            val largeUri = resizeAndSave(bitmap, 800, 800, "large_")
            setProgress(workDataOf("PROGRESS" to 100))
            
            Result.success(workDataOf(
                "THUMBNAIL_URI" to thumbnailUri,
                "MEDIUM_URI" to mediumUri,
                "LARGE_URI" to largeUri
            ))
        } catch (e: Exception) {
            Result.failure(workDataOf("ERROR" to e.message))
        }
    }
    
    private suspend fun loadImage(uri: String): Any {
        delay(500)  // simulate loading
        return Object()
    }
    
    private suspend fun resizeAndSave(
        bitmap: Any,
        width: Int,
        height: Int,
        prefix: String
    ): String {
        delay(300)  // simulate processing
        return "file:///$prefix${width}x${height}.jpg"
    }
}

// TODO: สร้าง UI ที่แสดง progress ของแต่ละ step
// TODO: แสดง preview ของทั้ง 3 ขนาดเมื่อเสร็จ
```

---

*Part 30 จบแล้ว | ก่อนหน้า: [Part 29](../part29/README.md) | ถัดไป: [Part 51](../part51/README.md)*
