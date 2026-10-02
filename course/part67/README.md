# Part 67: Push Notifications & Firebase
## ขั้นตอนที่ 1151-1175

---

## ขั้นตอนที่ 1151: Firebase Setup

```kotlin
// build.gradle.kts (project)
plugins {
    id("com.google.gms.google-services") version "4.x" apply false
    id("com.google.firebase.crashlytics") version "3.x" apply false
}

// build.gradle.kts (app)
plugins {
    id("com.google.gms.google-services")
    id("com.google.firebase.crashlytics")
}

dependencies {
    // Firebase BOM (Bill of Materials) - manage versions
    implementation(platform("com.google.firebase:firebase-bom:33.x"))
    
    implementation("com.google.firebase:firebase-analytics-ktx")
    implementation("com.google.firebase:firebase-crashlytics-ktx")
    implementation("com.google.firebase:firebase-messaging-ktx")
    implementation("com.google.firebase:firebase-config-ktx")
    implementation("com.google.firebase:firebase-auth-ktx")
    implementation("com.google.firebase:firebase-firestore-ktx")
}
```

---

## ขั้นตอนที่ 1152: FCM Push Notifications

```kotlin
// ============================================
// Firebase Cloud Messaging Service
// ============================================

@AndroidEntryPoint
class MyFirebaseMessagingService : FirebaseMessagingService() {
    
    @Inject
    lateinit var notificationManager: AppNotificationManager
    
    @Inject
    lateinit var tokenRepository: TokenRepository
    
    // เรียกเมื่อ token ใหม่ถูกสร้าง (ควร upload ไป server)
    override fun onNewToken(token: String) {
        super.onNewToken(token)
        
        // Upload token to backend
        CoroutineScope(Dispatchers.IO).launch {
            tokenRepository.updateFcmToken(token)
        }
    }
    
    // เรียกเมื่อได้รับ message
    override fun onMessageReceived(remoteMessage: RemoteMessage) {
        super.onMessageReceived(remoteMessage)
        
        // Data message (background/foreground)
        remoteMessage.data.isNotEmpty().let { hasData ->
            if (hasData) {
                handleDataMessage(remoteMessage.data)
            }
        }
        
        // Notification message (system tray เมื่อ app ใน background)
        remoteMessage.notification?.let { notification ->
            showNotification(
                title = notification.title ?: "New Notification",
                body = notification.body ?: "",
                data = remoteMessage.data
            )
        }
    }
    
    private fun handleDataMessage(data: Map<String, String>) {
        val type = data["type"] ?: return
        
        when (type) {
            "order_update" -> {
                val orderId = data["order_id"]
                val status = data["status"]
                showOrderUpdateNotification(orderId, status)
            }
            "promotion" -> {
                val title = data["title"] ?: "โปรโมชั่นพิเศษ"
                val body = data["body"] ?: ""
                val deepLink = data["deep_link"]
                showPromotionNotification(title, body, deepLink)
            }
            "chat" -> {
                val senderId = data["sender_id"]
                val message = data["message"]
                showChatNotification(senderId, message)
            }
        }
    }
    
    private fun showNotification(title: String, body: String, data: Map<String, String> = emptyMap()) {
        notificationManager.show(
            NotificationData(
                id = System.currentTimeMillis().toInt(),
                title = title,
                body = body,
                channelId = GENERAL_CHANNEL_ID,
                data = data
            )
        )
    }
    
    private fun showOrderUpdateNotification(orderId: String?, status: String?) {
        notificationManager.show(
            NotificationData(
                id = orderId.hashCode(),
                title = "อัปเดตคำสั่งซื้อ",
                body = when (status) {
                    "shipped" -> "สินค้าของคุณถูกจัดส่งแล้ว"
                    "delivered" -> "สินค้าถึงปลายทางแล้ว"
                    else -> "คำสั่งซื้อ #$orderId มีการอัปเดต"
                },
                channelId = ORDERS_CHANNEL_ID,
                deepLink = "app://orders/$orderId"
            )
        )
    }
    
    companion object {
        const val GENERAL_CHANNEL_ID = "general"
        const val ORDERS_CHANNEL_ID = "orders"
        const val PROMOTIONS_CHANNEL_ID = "promotions"
        const val CHAT_CHANNEL_ID = "chat"
    }
}

// ============================================
// Notification Manager
// ============================================

class AppNotificationManager @Inject constructor(
    @ApplicationContext private val context: Context
) {
    
    private val notificationManager = NotificationManagerCompat.from(context)
    
    init {
        createChannels()
    }
    
    fun show(data: NotificationData) {
        val pendingIntent = data.deepLink?.let { deepLink ->
            val intent = Intent(Intent.ACTION_VIEW, Uri.parse(deepLink)).apply {
                addFlags(Intent.FLAG_ACTIVITY_CLEAR_TOP)
            }
            PendingIntent.getActivity(
                context, data.id, intent,
                PendingIntent.FLAG_ONE_SHOT or PendingIntent.FLAG_IMMUTABLE
            )
        }
        
        val notification = NotificationCompat.Builder(context, data.channelId)
            .setContentTitle(data.title)
            .setContentText(data.body)
            .setSmallIcon(R.drawable.ic_notification)
            .setAutoCancel(true)
            .setPriority(NotificationCompat.PRIORITY_DEFAULT)
            .apply {
                pendingIntent?.let { setContentIntent(it) }
                data.imageUrl?.let { url ->
                    // Load image for big picture style
                    // Coil/Glide สามารถโหลด URL ได้
                }
            }
            .build()
        
        if (ActivityCompat.checkSelfPermission(
                context, Manifest.permission.POST_NOTIFICATIONS
            ) == PackageManager.PERMISSION_GRANTED
        ) {
            notificationManager.notify(data.id, notification)
        }
    }
    
    private fun createChannels() {
        val channels = listOf(
            NotificationChannel(
                MyFirebaseMessagingService.GENERAL_CHANNEL_ID,
                "ทั่วไป",
                NotificationManager.IMPORTANCE_DEFAULT
            ),
            NotificationChannel(
                MyFirebaseMessagingService.ORDERS_CHANNEL_ID,
                "คำสั่งซื้อ",
                NotificationManager.IMPORTANCE_HIGH
            ).apply {
                description = "อัปเดตสถานะคำสั่งซื้อ"
            },
            NotificationChannel(
                MyFirebaseMessagingService.PROMOTIONS_CHANNEL_ID,
                "โปรโมชั่น",
                NotificationManager.IMPORTANCE_LOW
            ),
            NotificationChannel(
                MyFirebaseMessagingService.CHAT_CHANNEL_ID,
                "ข้อความ",
                NotificationManager.IMPORTANCE_HIGH
            )
        )
        
        val manager = context.getSystemService(NotificationManager::class.java)
        channels.forEach { manager.createNotificationChannel(it) }
    }
}

data class NotificationData(
    val id: Int,
    val title: String,
    val body: String,
    val channelId: String,
    val deepLink: String? = null,
    val imageUrl: String? = null,
    val data: Map<String, String> = emptyMap()
)
```

---

## ขั้นตอนที่ 1153: Firebase Crashlytics

```kotlin
// ============================================
// Crashlytics Setup
// ============================================

// Application class
class MyApplication : Application() {
    override fun onCreate() {
        super.onCreate()
        
        // Enable/disable based on environment
        FirebaseCrashlytics.getInstance().setCrashlyticsCollectionEnabled(
            !BuildConfig.DEBUG
        )
    }
}

// ============================================
// Custom Error Reporting
// ============================================

class CrashReporter @Inject constructor() {
    
    fun recordException(throwable: Throwable) {
        FirebaseCrashlytics.getInstance().recordException(throwable)
    }
    
    fun setUserIdentifier(userId: String) {
        FirebaseCrashlytics.getInstance().setUserId(userId)
    }
    
    fun log(message: String) {
        FirebaseCrashlytics.getInstance().log(message)
    }
    
    fun setCustomKey(key: String, value: String) {
        FirebaseCrashlytics.getInstance().setCustomKey(key, value)
    }
    
    fun setCustomKey(key: String, value: Boolean) {
        FirebaseCrashlytics.getInstance().setCustomKey(key, value)
    }
}

// Global exception handler
class GlobalExceptionHandler @Inject constructor(
    private val crashReporter: CrashReporter
) : Thread.UncaughtExceptionHandler {
    
    private val defaultHandler = Thread.getDefaultUncaughtExceptionHandler()
    
    override fun uncaughtException(thread: Thread, throwable: Throwable) {
        crashReporter.log("Uncaught exception on thread: ${thread.name}")
        crashReporter.recordException(throwable)
        defaultHandler?.uncaughtException(thread, throwable)
    }
}

// ใน ViewModel - catch และ report
@HiltViewModel
class ProductViewModel @Inject constructor(
    private val repository: ProductRepository,
    private val crashReporter: CrashReporter
) : ViewModel() {
    
    fun loadProduct(id: Long) {
        viewModelScope.launch {
            try {
                val product = repository.getProduct(id)
                // handle success
            } catch (e: Exception) {
                crashReporter.log("Failed to load product $id")
                crashReporter.recordException(e)
                // handle error in UI
            }
        }
    }
}
```

---

## แบบฝึกหัด Part 67

```kotlin
// แบบฝึกหัด: Notification System

// TODO: สร้าง NotificationViewModel ที่:
// 1. Request notification permission (Android 13+)
// 2. Subscribe/unsubscribe FCM topics
// 3. แสดงรายการ notifications ใน-app
// 4. Mark as read
// 5. Deep link navigation เมื่อกด notification

@HiltViewModel
class NotificationViewModel @Inject constructor(
    private val notificationRepository: NotificationRepository,
    private val fcmManager: FcmManager
) : ViewModel() {
    
    // TODO: implement
    
    fun requestPermission(launcher: ActivityResultLauncher<String>) {
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.TIRAMISU) {
            launcher.launch(Manifest.permission.POST_NOTIFICATIONS)
        }
    }
    
    fun subscribeToTopic(topic: String) {
        // TODO: FirebaseMessaging.getInstance().subscribeToTopic(topic)
    }
    
    fun markAsRead(notificationId: String) {
        // TODO: update in DB
    }
}
```

---

*Part 67 จบแล้ว | ก่อนหน้า: [Part 66](../part66/README.md) | ถัดไป: [Part 68](../part68/README.md)*
