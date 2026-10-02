# Part 94: Firebase Advanced Integration
## ขั้นตอนที่ 1826-1850

---

## ขั้นตอนที่ 1826: Firestore Real-Time Sync

```kotlin
// ============================================
// Firestore - Real-time Database
// ============================================

// Data models
@Serializable
data class ChatRoom(
    val id: String = "",
    val name: String = "",
    val participants: List<String> = emptyList(),
    val lastMessage: String = "",
    val lastMessageAt: Long = 0L,
    val createdAt: Long = System.currentTimeMillis()
)

@Serializable
data class Message(
    val id: String = "",
    val roomId: String = "",
    val senderId: String = "",
    val senderName: String = "",
    val content: String = "",
    val imageUrl: String? = null,
    val timestamp: Long = System.currentTimeMillis(),
    val readBy: List<String> = emptyList()
)

// Firestore Repository
class FirestoreChatRepository @Inject constructor(
    private val firestore: FirebaseFirestore,
    private val auth: FirebaseAuth
) {
    
    private val currentUserId get() = auth.currentUser?.uid ?: error("Not authenticated")
    
    // Observe chat rooms in real-time
    fun observeMyRooms(): Flow<List<ChatRoom>> = callbackFlow {
        val listener = firestore
            .collection("rooms")
            .whereArrayContains("participants", currentUserId)
            .orderBy("lastMessageAt", Query.Direction.DESCENDING)
            .addSnapshotListener { snapshot, error ->
                if (error != null) {
                    close(error)
                    return@addSnapshotListener
                }
                
                val rooms = snapshot?.documents?.mapNotNull { doc ->
                    doc.toObject(ChatRoom::class.java)?.copy(id = doc.id)
                } ?: emptyList()
                
                trySend(rooms)
            }
        
        awaitClose { listener.remove() }
    }.flowOn(Dispatchers.IO)
    
    // Observe messages in real-time
    fun observeMessages(roomId: String, limit: Long = 50): Flow<List<Message>> = callbackFlow {
        val listener = firestore
            .collection("rooms")
            .document(roomId)
            .collection("messages")
            .orderBy("timestamp", Query.Direction.DESCENDING)
            .limit(limit)
            .addSnapshotListener { snapshot, error ->
                if (error != null) {
                    close(error)
                    return@addSnapshotListener
                }
                
                val messages = snapshot?.documents?.mapNotNull { doc ->
                    doc.toObject(Message::class.java)?.copy(id = doc.id)
                }?.reversed() ?: emptyList()
                
                trySend(messages)
            }
        
        awaitClose { listener.remove() }
    }.flowOn(Dispatchers.IO)
    
    // Send message
    suspend fun sendMessage(roomId: String, content: String, imageUrl: String? = null) {
        val message = Message(
            roomId = roomId,
            senderId = currentUserId,
            senderName = auth.currentUser?.displayName ?: "Unknown",
            content = content,
            imageUrl = imageUrl
        )
        
        // Use batch write for atomicity
        firestore.runBatch { batch ->
            // Add message
            val messageRef = firestore
                .collection("rooms")
                .document(roomId)
                .collection("messages")
                .document()
            batch.set(messageRef, message.copy(id = messageRef.id))
            
            // Update room's last message
            val roomRef = firestore.collection("rooms").document(roomId)
            batch.update(roomRef, mapOf(
                "lastMessage" to content,
                "lastMessageAt" to message.timestamp
            ))
        }.await()
    }
    
    // Mark messages as read
    suspend fun markAsRead(roomId: String, messageIds: List<String>) {
        firestore.runBatch { batch ->
            messageIds.forEach { messageId ->
                val ref = firestore
                    .collection("rooms")
                    .document(roomId)
                    .collection("messages")
                    .document(messageId)
                batch.update(ref, "readBy", FieldValue.arrayUnion(currentUserId))
            }
        }.await()
    }
    
    // Create room
    suspend fun createRoom(participantIds: List<String>, name: String): String {
        val room = ChatRoom(
            name = name,
            participants = participantIds + currentUserId
        )
        
        val docRef = firestore.collection("rooms").add(room).await()
        return docRef.id
    }
}
```

---

## ขั้นตอนที่ 1827: Firebase Auth

```kotlin
// ============================================
// Firebase Authentication
// ============================================

class AuthRepository @Inject constructor(
    private val auth: FirebaseAuth,
    private val googleSignInClient: GoogleSignInClient
) {
    
    val currentUser: FirebaseUser? get() = auth.currentUser
    
    // Auth state as Flow
    val authState: Flow<FirebaseUser?> = callbackFlow {
        val listener = FirebaseAuth.AuthStateListener { firebaseAuth ->
            trySend(firebaseAuth.currentUser)
        }
        auth.addAuthStateListener(listener)
        awaitClose { auth.removeAuthStateListener(listener) }
    }
    
    // Email/Password sign in
    suspend fun signInWithEmail(email: String, password: String): AuthResult {
        return try {
            auth.signInWithEmailAndPassword(email, password).await()
            AuthResult.Success
        } catch (e: FirebaseAuthException) {
            AuthResult.Error(mapFirebaseError(e))
        }
    }
    
    // Google Sign In
    fun getGoogleSignInIntent(): Intent = googleSignInClient.signInIntent
    
    suspend fun handleGoogleSignInResult(idToken: String): AuthResult {
        return try {
            val credential = GoogleAuthProvider.getCredential(idToken, null)
            auth.signInWithCredential(credential).await()
            AuthResult.Success
        } catch (e: Exception) {
            AuthResult.Error(AuthError.GOOGLE_SIGN_IN_FAILED)
        }
    }
    
    // Create account
    suspend fun createAccount(email: String, password: String, displayName: String): AuthResult {
        return try {
            val result = auth.createUserWithEmailAndPassword(email, password).await()
            
            // Update display name
            result.user?.updateProfile(
                UserProfileChangeRequest.Builder()
                    .setDisplayName(displayName)
                    .build()
            )?.await()
            
            // Send email verification
            result.user?.sendEmailVerification()?.await()
            
            AuthResult.Success
        } catch (e: FirebaseAuthException) {
            AuthResult.Error(mapFirebaseError(e))
        }
    }
    
    // Password reset
    suspend fun sendPasswordReset(email: String): Result<Unit> {
        return try {
            auth.sendPasswordResetEmail(email).await()
            Result.success(Unit)
        } catch (e: Exception) {
            Result.failure(e)
        }
    }
    
    fun signOut() {
        auth.signOut()
        googleSignInClient.signOut()
    }
    
    private fun mapFirebaseError(e: FirebaseAuthException): AuthError = when (e.errorCode) {
        "ERROR_INVALID_EMAIL" -> AuthError.INVALID_EMAIL
        "ERROR_WRONG_PASSWORD" -> AuthError.WRONG_PASSWORD
        "ERROR_USER_NOT_FOUND" -> AuthError.USER_NOT_FOUND
        "ERROR_USER_DISABLED" -> AuthError.USER_DISABLED
        "ERROR_EMAIL_ALREADY_IN_USE" -> AuthError.EMAIL_ALREADY_EXISTS
        "ERROR_WEAK_PASSWORD" -> AuthError.WEAK_PASSWORD
        "ERROR_TOO_MANY_REQUESTS" -> AuthError.TOO_MANY_ATTEMPTS
        else -> AuthError.UNKNOWN
    }
}

sealed class AuthResult {
    object Success : AuthResult()
    data class Error(val error: AuthError) : AuthResult()
}

enum class AuthError {
    INVALID_EMAIL, WRONG_PASSWORD, USER_NOT_FOUND, USER_DISABLED,
    EMAIL_ALREADY_EXISTS, WEAK_PASSWORD, TOO_MANY_ATTEMPTS,
    GOOGLE_SIGN_IN_FAILED, UNKNOWN
}
```

---

## ขั้นตอนที่ 1828: Firebase Remote Config

```kotlin
// ============================================
// Remote Config - Feature Flags & A/B Testing
// ============================================

class RemoteConfigManager @Inject constructor(
    private val remoteConfig: FirebaseRemoteConfig
) {
    
    init {
        // Set defaults
        remoteConfig.setDefaultsAsync(R.xml.remote_config_defaults)
        
        // Development: real-time updates every 5 seconds
        // Production: 12 hours (default)
        val settings = if (BuildConfig.DEBUG) {
            FirebaseRemoteConfigSettings.Builder()
                .setMinimumFetchIntervalInSeconds(5)
                .build()
        } else {
            FirebaseRemoteConfigSettings.Builder()
                .setMinimumFetchIntervalInSeconds(43200) // 12 hours
                .build()
        }
        remoteConfig.setConfigSettingsAsync(settings)
    }
    
    // Feature flags
    val isNewCheckoutEnabled: Boolean
        get() = remoteConfig.getBoolean("feature_new_checkout")
    
    val isReferralProgramEnabled: Boolean
        get() = remoteConfig.getBoolean("feature_referral_program")
    
    val homeBannerVersion: String
        get() = remoteConfig.getString("home_banner_version")
    
    // A/B test values
    val checkoutButtonColor: String
        get() = remoteConfig.getString("ab_checkout_button_color") // "red" or "blue"
    
    val recommendationsAlgorithm: String
        get() = remoteConfig.getString("recommendations_algorithm") // "v1", "v2", "collaborative"
    
    // Fetch and activate
    suspend fun fetchAndActivate(): Boolean {
        return try {
            remoteConfig.fetchAndActivate().await()
        } catch (e: Exception) {
            false  // Use defaults on failure
        }
    }
    
    // Observe value changes (for real-time updates)
    fun observeFeatureFlag(key: String): Flow<Boolean> = callbackFlow {
        remoteConfig.addOnConfigUpdateListener(object : ConfigUpdateListener {
            override fun onUpdate(configUpdate: ConfigUpdate) {
                if (key in configUpdate.updatedKeys) {
                    remoteConfig.activate().addOnCompleteListener {
                        trySend(remoteConfig.getBoolean(key))
                    }
                }
            }
            override fun onError(error: FirebaseRemoteConfigException) {
                // Handle error
            }
        })
        
        // Emit current value immediately
        trySend(remoteConfig.getBoolean(key))
        
        awaitClose()
    }
}

// res/xml/remote_config_defaults.xml
/*
<?xml version="1.0" encoding="utf-8"?>
<defaultsMap>
    <entry>
        <key>feature_new_checkout</key>
        <value>false</value>
    </entry>
    <entry>
        <key>feature_referral_program</key>
        <value>false</value>
    </entry>
    <entry>
        <key>home_banner_version</key>
        <value>v1</value>
    </entry>
    <entry>
        <key>ab_checkout_button_color</key>
        <value>blue</value>
    </entry>
</defaultsMap>
*/
```

---

## ขั้นตอนที่ 1829: Firebase Cloud Messaging

```kotlin
// ============================================
// FCM - Push Notifications
// ============================================

@AndroidEntryPoint
class MyFirebaseMessagingService : FirebaseMessagingService() {
    
    @Inject
    lateinit var notificationManager: AppNotificationManager
    
    @Inject
    lateinit var tokenRepository: FcmTokenRepository
    
    override fun onNewToken(token: String) {
        // Register token with backend
        CoroutineScope(Dispatchers.IO).launch {
            tokenRepository.saveToken(token)
        }
    }
    
    override fun onMessageReceived(remoteMessage: RemoteMessage) {
        val type = remoteMessage.data["type"] ?: "general"
        val title = remoteMessage.notification?.title ?: remoteMessage.data["title"] ?: ""
        val body = remoteMessage.notification?.body ?: remoteMessage.data["body"] ?: ""
        
        when (type) {
            "order_update" -> {
                val orderId = remoteMessage.data["orderId"] ?: ""
                notificationManager.showOrderUpdate(orderId, title, body)
            }
            "chat_message" -> {
                val roomId = remoteMessage.data["roomId"] ?: ""
                val senderId = remoteMessage.data["senderId"] ?: ""
                notificationManager.showChatNotification(roomId, senderId, title, body)
            }
            else -> {
                notificationManager.showGeneral(title, body)
            }
        }
    }
}

class AppNotificationManager @Inject constructor(
    private val context: Context,
    private val notificationManagerCompat: NotificationManagerCompat
) {
    
    companion object {
        const val CHANNEL_ORDERS = "orders"
        const val CHANNEL_CHAT = "chat"
        const val CHANNEL_GENERAL = "general"
    }
    
    fun createChannels() {
        val channels = listOf(
            NotificationChannelCompat.Builder(CHANNEL_ORDERS, NotificationManagerCompat.IMPORTANCE_HIGH)
                .setName("คำสั่งซื้อ")
                .setDescription("การอัปเดตสถานะคำสั่งซื้อ")
                .build(),
            
            NotificationChannelCompat.Builder(CHANNEL_CHAT, NotificationManagerCompat.IMPORTANCE_DEFAULT)
                .setName("ข้อความ")
                .setDescription("ข้อความในห้องสนทนา")
                .build(),
            
            NotificationChannelCompat.Builder(CHANNEL_GENERAL, NotificationManagerCompat.IMPORTANCE_LOW)
                .setName("ทั่วไป")
                .setDescription("การแจ้งเตือนทั่วไป")
                .build()
        )
        
        notificationManagerCompat.createNotificationChannelsCompat(channels)
    }
    
    @SuppressLint("MissingPermission")
    fun showOrderUpdate(orderId: String, title: String, body: String) {
        val intent = Intent(context, MainActivity::class.java).apply {
            putExtra("orderId", orderId)
            flags = Intent.FLAG_ACTIVITY_SINGLE_TOP
        }
        
        val pendingIntent = PendingIntent.getActivity(
            context, orderId.hashCode(), intent,
            PendingIntent.FLAG_IMMUTABLE or PendingIntent.FLAG_UPDATE_CURRENT
        )
        
        val notification = NotificationCompat.Builder(context, CHANNEL_ORDERS)
            .setSmallIcon(R.drawable.ic_notification)
            .setContentTitle(title)
            .setContentText(body)
            .setStyle(NotificationCompat.BigTextStyle().bigText(body))
            .setContentIntent(pendingIntent)
            .setAutoCancel(true)
            .build()
        
        notificationManagerCompat.notify(orderId.hashCode(), notification)
    }
}
```

---

*Part 94 จบแล้ว | ก่อนหน้า: [Part 93](../part93/README.md) | ถัดไป: [Part 95](../part95/README.md)*
