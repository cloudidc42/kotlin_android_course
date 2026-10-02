# Part 48: Android - Notifications พื้นฐาน
## ขั้นตอนที่ 976-1000

---

## ขั้นตอนที่ 976: ตั้งค่า Notification Channel (Android 8+)

Notification Channel จำเป็นสำหรับ Android 8.0 (API 26) ขึ้นไป แบ่ง notification เป็นกลุ่ม

```kotlin
import android.app.NotificationChannel
import android.app.NotificationManager
import android.content.Context
import android.os.Build

object NotificationChannels {
    const val GENERAL = "general_channel"
    const val MESSAGES = "messages_channel"
    const val PROMOTIONS = "promotions_channel"
    const val SILENT = "silent_channel"
}

fun Context.createNotificationChannels() {
    if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.O) {
        val notificationManager = getSystemService(NotificationManager::class.java)

        // Channel ทั่วไป
        val generalChannel = NotificationChannel(
            NotificationChannels.GENERAL,
            "ทั่วไป",
            NotificationManager.IMPORTANCE_DEFAULT
        ).apply {
            description = "การแจ้งเตือนทั่วไปจากแอป"
            enableLights(true)
            lightColor = android.graphics.Color.BLUE
            enableVibration(true)
        }

        // Channel ข้อความ (สำคัญ)
        val messagesChannel = NotificationChannel(
            NotificationChannels.MESSAGES,
            "ข้อความ",
            NotificationManager.IMPORTANCE_HIGH
        ).apply {
            description = "ข้อความจากผู้ใช้งานอื่น"
            enableLights(true)
            lightColor = android.graphics.Color.GREEN
            setShowBadge(true)
        }

        // Channel โปรโมชัน (ต่ำ)
        val promotionsChannel = NotificationChannel(
            NotificationChannels.PROMOTIONS,
            "โปรโมชัน",
            NotificationManager.IMPORTANCE_LOW
        ).apply {
            description = "โปรโมชันและข้อเสนอพิเศษ"
        }

        // Channel เงียบ
        val silentChannel = NotificationChannel(
            NotificationChannels.SILENT,
            "เงียบ",
            NotificationManager.IMPORTANCE_MIN
        )

        notificationManager.createNotificationChannels(
            listOf(generalChannel, messagesChannel, promotionsChannel, silentChannel)
        )
    }
}

// เรียกใน Application.onCreate()
class MyApplication : Application() {
    override fun onCreate() {
        super.onCreate()
        createNotificationChannels()
    }
}
```

---

## ขั้นตอนที่ 977: สร้าง Notification พื้นฐาน

```kotlin
import android.app.PendingIntent
import android.content.Intent
import androidx.core.app.NotificationCompat
import androidx.core.app.NotificationManagerCompat

class NotificationHelper(private val context: Context) {

    private val notificationManager = NotificationManagerCompat.from(context)

    // Notification พื้นฐาน
    fun showSimpleNotification(
        id: Int,
        title: String,
        message: String
    ) {
        // ตรวจสอบ permission (Android 13+)
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.TIRAMISU) {
            if (ContextCompat.checkSelfPermission(
                    context,
                    Manifest.permission.POST_NOTIFICATIONS
                ) != PackageManager.PERMISSION_GRANTED
            ) return
        }

        val notification = NotificationCompat.Builder(context, NotificationChannels.GENERAL)
            .setSmallIcon(R.drawable.ic_notification)
            .setContentTitle(title)
            .setContentText(message)
            .setPriority(NotificationCompat.PRIORITY_DEFAULT)
            .setAutoCancel(true) // ลบเองเมื่อกด
            .build()

        notificationManager.notify(id, notification)
    }

    // Notification พร้อม PendingIntent (คลิกเพื่อเปิด activity)
    fun showActionableNotification(id: Int, title: String, message: String, itemId: Int) {
        val intent = Intent(context, MainActivity::class.java).apply {
            flags = Intent.FLAG_ACTIVITY_NEW_TASK or Intent.FLAG_ACTIVITY_CLEAR_TASK
            putExtra("item_id", itemId)
        }

        val pendingIntent = PendingIntent.getActivity(
            context, id, intent,
            PendingIntent.FLAG_UPDATE_CURRENT or PendingIntent.FLAG_IMMUTABLE
        )

        val notification = NotificationCompat.Builder(context, NotificationChannels.MESSAGES)
            .setSmallIcon(R.drawable.ic_notification)
            .setContentTitle(title)
            .setContentText(message)
            .setPriority(NotificationCompat.PRIORITY_HIGH)
            .setContentIntent(pendingIntent)
            .setAutoCancel(true)
            // Action buttons
            .addAction(
                R.drawable.ic_reply,
                "ตอบกลับ",
                createReplyPendingIntent(id)
            )
            .addAction(
                R.drawable.ic_mark_read,
                "ทำเครื่องหมายว่าอ่านแล้ว",
                createMarkReadPendingIntent(id)
            )
            .build()

        notificationManager.notify(id, notification)
    }

    private fun createReplyPendingIntent(notificationId: Int): PendingIntent {
        val intent = Intent(context, NotificationActionReceiver::class.java).apply {
            action = "ACTION_REPLY"
            putExtra("notification_id", notificationId)
        }
        return PendingIntent.getBroadcast(
            context, notificationId, intent,
            PendingIntent.FLAG_UPDATE_CURRENT or PendingIntent.FLAG_IMMUTABLE
        )
    }

    private fun createMarkReadPendingIntent(notificationId: Int): PendingIntent {
        val intent = Intent(context, NotificationActionReceiver::class.java).apply {
            action = "ACTION_MARK_READ"
            putExtra("notification_id", notificationId)
        }
        return PendingIntent.getBroadcast(
            context, notificationId, intent,
            PendingIntent.FLAG_UPDATE_CURRENT or PendingIntent.FLAG_IMMUTABLE
        )
    }

    fun cancelNotification(id: Int) {
        notificationManager.cancel(id)
    }

    fun cancelAllNotifications() {
        notificationManager.cancelAll()
    }
}
```

---

## ขั้นตอนที่ 978: Notification พร้อม Big Picture และ Inbox Style

```kotlin
fun showBigPictureNotification(id: Int) {
    val bigPicture = BitmapFactory.decodeResource(context.resources, R.drawable.notification_image)

    val notification = NotificationCompat.Builder(context, NotificationChannels.GENERAL)
        .setSmallIcon(R.drawable.ic_notification)
        .setContentTitle("รูปใหม่จากเพื่อน")
        .setContentText("มานี ส่งรูปให้คุณ")
        .setLargeIcon(bigPicture)
        .setStyle(
            NotificationCompat.BigPictureStyle()
                .bigPicture(bigPicture)
                .setBigContentTitle("รูปจากมานี")
                .setSummaryText("คลิกเพื่อดูรูปเต็ม")
        )
        .setPriority(NotificationCompat.PRIORITY_DEFAULT)
        .setAutoCancel(true)
        .build()

    notificationManager.notify(id, notification)
}

fun showInboxStyleNotification(id: Int, messages: List<String>) {
    val inboxStyle = NotificationCompat.InboxStyle()

    messages.forEach { message ->
        inboxStyle.addLine(message)
    }
    inboxStyle.setSummaryText("${messages.size} ข้อความใหม่")

    val notification = NotificationCompat.Builder(context, NotificationChannels.MESSAGES)
        .setSmallIcon(R.drawable.ic_notification)
        .setContentTitle("ข้อความใหม่")
        .setContentText("${messages.size} ข้อความที่ยังไม่ได้อ่าน")
        .setStyle(inboxStyle)
        .setPriority(NotificationCompat.PRIORITY_HIGH)
        .setNumber(messages.size)
        .setAutoCancel(true)
        .build()

    notificationManager.notify(id, notification)
}

fun showBigTextNotification(id: Int, title: String, longText: String) {
    val notification = NotificationCompat.Builder(context, NotificationChannels.GENERAL)
        .setSmallIcon(R.drawable.ic_notification)
        .setContentTitle(title)
        .setContentText(longText.take(100))
        .setStyle(
            NotificationCompat.BigTextStyle()
                .bigText(longText)
                .setBigContentTitle(title)
        )
        .setPriority(NotificationCompat.PRIORITY_DEFAULT)
        .setAutoCancel(true)
        .build()

    notificationManager.notify(id, notification)
}
```

---

## ขั้นตอนที่ 979: Progress Notification

```kotlin
fun showProgressNotification(id: Int, title: String) {
    val PROGRESS_MAX = 100
    var currentProgress = 0

    // เริ่ม progress
    val builder = NotificationCompat.Builder(context, NotificationChannels.GENERAL)
        .setSmallIcon(R.drawable.ic_download)
        .setContentTitle(title)
        .setContentText("กำลังดาวน์โหลด...")
        .setPriority(NotificationCompat.PRIORITY_LOW)
        .setOngoing(true) // ผู้ใช้ไม่สามารถปิดเองได้
        .setProgress(PROGRESS_MAX, currentProgress, false)

    notificationManager.notify(id, builder.build())

    // อัพเดท progress ในแต่ละ step
    CoroutineScope(Dispatchers.IO).launch {
        while (currentProgress < PROGRESS_MAX) {
            delay(500)
            currentProgress += 10
            builder.setProgress(PROGRESS_MAX, currentProgress, false)
                .setContentText("ดาวน์โหลด $currentProgress%")
            notificationManager.notify(id, builder.build())
        }

        // เสร็จแล้ว
        builder.setContentText("ดาวน์โหลดสำเร็จ!")
            .setProgress(0, 0, false)
            .setOngoing(false)
            .setAutoCancel(true)
        notificationManager.notify(id, builder.build())
    }
}

// Indeterminate progress
fun showIndeterminateProgressNotification(id: Int, title: String) {
    val notification = NotificationCompat.Builder(context, NotificationChannels.GENERAL)
        .setSmallIcon(R.drawable.ic_sync)
        .setContentTitle(title)
        .setContentText("กำลังประมวลผล...")
        .setPriority(NotificationCompat.PRIORITY_LOW)
        .setOngoing(true)
        .setProgress(0, 0, true) // indeterminate = true
        .build()

    notificationManager.notify(id, notification)
}
```

---

## ขั้นตอนที่ 980: ขอสิทธิ์ POST_NOTIFICATIONS (Android 13+)

```kotlin
import android.Manifest
import androidx.activity.compose.rememberLauncherForActivityResult
import androidx.activity.result.contract.ActivityResultContracts

@Composable
fun NotificationPermissionScreen() {
    val context = LocalContext.current
    var hasPermission by remember {
        mutableStateOf(
            if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.TIRAMISU) {
                ContextCompat.checkSelfPermission(
                    context, Manifest.permission.POST_NOTIFICATIONS
                ) == PackageManager.PERMISSION_GRANTED
            } else true
        )
    }

    val notificationPermissionLauncher = rememberLauncherForActivityResult(
        ActivityResultContracts.RequestPermission()
    ) { isGranted ->
        hasPermission = isGranted
    }

    Column(modifier = Modifier.padding(16.dp)) {
        if (hasPermission) {
            Button(onClick = {
                val helper = NotificationHelper(context)
                helper.showSimpleNotification(
                    id = 1,
                    title = "ทดสอบการแจ้งเตือน",
                    message = "Notification ทำงานแล้ว!"
                )
            }) {
                Text("ส่ง Notification ทดสอบ")
            }

            Button(onClick = {
                val helper = NotificationHelper(context)
                helper.showInboxStyleNotification(
                    id = 2,
                    messages = listOf("มานี: สวัสดี!", "สมชาย: โอเค", "วิชัย: ตกลง")
                )
            }) {
                Text("ส่ง Inbox Notification")
            }
        } else {
            Column {
                Text("แอปต้องการสิทธิ์ส่งการแจ้งเตือน")
                Button(onClick = {
                    if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.TIRAMISU) {
                        notificationPermissionLauncher.launch(Manifest.permission.POST_NOTIFICATIONS)
                    }
                }) {
                    Text("อนุญาตการแจ้งเตือน")
                }
            }
        }
    }
}
```

---

*Part 48 จบแล้ว | ก่อนหน้า: [Part 47](../part47/README.md) | ถัดไป: [Part 49](../part49/README.md)*
