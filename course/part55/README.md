# Part 55: App Security
## ขั้นตอนที่ 851-875

---

## ขั้นตอนที่ 851: Security Fundamentals

```
Android Security Layers:
┌─────────────────────────────────────┐
│         Application Layer           │
│  (Network security, data encryption)│
├─────────────────────────────────────┤
│         Android Framework           │
│  (Permissions, IPC, Sandboxing)     │
├─────────────────────────────────────┤
│           Linux Kernel              │
│  (Process isolation, file system)   │
└─────────────────────────────────────┘

หัวข้อหลัก:
1. Network Security (HTTPS, certificate pinning)
2. Data Security (encryption, secure storage)
3. Authentication (biometric, OAuth)
4. Code Security (ProGuard/R8, obfuscation)
5. Secure Communication (IPC, Intent security)
```

---

## ขั้นตอนที่ 852: Network Security Configuration

```xml
<!-- res/xml/network_security_config.xml -->
<?xml version="1.0" encoding="utf-8"?>
<network-security-config>
    <!-- กำหนด config เฉพาะ production domain -->
    <domain-config cleartextTrafficPermitted="false">
        <domain includeSubdomains="true">api.example.com</domain>
        
        <!-- Certificate Pinning -->
        <pin-set expiration="2027-01-01">
            <!-- SHA-256 hash ของ certificate public key -->
            <pin digest="SHA-256">base64encodedHashHere=</pin>
            <!-- Backup pin -->
            <pin digest="SHA-256">backupBase64encodedHashHere=</pin>
        </pin-set>
    </domain-config>
    
    <!-- Debug builds อนุญาต cleartext -->
    <debug-overrides>
        <trust-anchors>
            <certificates src="user"/>
        </trust-anchors>
    </debug-overrides>
    
    <!-- Default: ไม่อนุญาต cleartext traffic -->
    <base-config cleartextTrafficPermitted="false">
        <trust-anchors>
            <certificates src="system"/>
        </trust-anchors>
    </base-config>
</network-security-config>
```

```kotlin
// AndroidManifest.xml
// android:networkSecurityConfig="@xml/network_security_config"

// Certificate Pinning ด้วย OkHttp
fun createSecureOkHttpClient(): OkHttpClient {
    return OkHttpClient.Builder()
        .certificatePinner(
            CertificatePinner.Builder()
                .add("api.example.com", "sha256/AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA=")
                .add("api.example.com", "sha256/BBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBB=")
                .build()
        )
        .connectTimeout(30, TimeUnit.SECONDS)
        .readTimeout(30, TimeUnit.SECONDS)
        .build()
}
```

---

## ขั้นตอนที่ 853: Encrypted Storage

```kotlin
// ============================================
// EncryptedSharedPreferences
// ============================================

// implementation("androidx.security:security-crypto:1.1.x")

fun createEncryptedPrefs(context: Context): SharedPreferences {
    val masterKey = MasterKey.Builder(context)
        .setKeyScheme(MasterKey.KeyScheme.AES256_GCM)
        .build()
    
    return EncryptedSharedPreferences.create(
        context,
        "secure_prefs",
        masterKey,
        EncryptedSharedPreferences.PrefKeyEncryptionScheme.AES256_SIV,
        EncryptedSharedPreferences.PrefValueEncryptionScheme.AES256_GCM
    )
}

// ============================================
// EncryptedFile
// ============================================

fun writeEncryptedFile(context: Context, data: String) {
    val masterKey = MasterKey.Builder(context)
        .setKeyScheme(MasterKey.KeyScheme.AES256_GCM)
        .build()
    
    val file = File(context.filesDir, "secret.dat")
    
    val encryptedFile = EncryptedFile.Builder(
        context,
        file,
        masterKey,
        EncryptedFile.FileEncryptionScheme.AES256_GCM_HKDF_4KB
    ).build()
    
    encryptedFile.openFileOutput().use { output ->
        output.write(data.toByteArray(Charsets.UTF_8))
    }
}

fun readEncryptedFile(context: Context): String {
    val masterKey = MasterKey.Builder(context)
        .setKeyScheme(MasterKey.KeyScheme.AES256_GCM)
        .build()
    
    val file = File(context.filesDir, "secret.dat")
    
    val encryptedFile = EncryptedFile.Builder(
        context,
        file,
        masterKey,
        EncryptedFile.FileEncryptionScheme.AES256_GCM_HKDF_4KB
    ).build()
    
    return encryptedFile.openFileInput().bufferedReader().readText()
}

// ============================================
// Android Keystore - เก็บ Key อย่างปลอดภัย
// ============================================

object KeystoreManager {
    private const val KEY_ALIAS = "my_secure_key"
    
    fun generateKey() {
        val keyGenerator = KeyGenerator.getInstance(
            KeyProperties.KEY_ALGORITHM_AES,
            "AndroidKeyStore"
        )
        
        val keyGenSpec = KeyGenParameterSpec.Builder(
            KEY_ALIAS,
            KeyProperties.PURPOSE_ENCRYPT or KeyProperties.PURPOSE_DECRYPT
        )
            .setBlockModes(KeyProperties.BLOCK_MODE_GCM)
            .setEncryptionPaddings(KeyProperties.ENCRYPTION_PADDING_NONE)
            .setUserAuthenticationRequired(false)
            .build()
        
        keyGenerator.init(keyGenSpec)
        keyGenerator.generateKey()
    }
    
    fun encrypt(data: String): Pair<ByteArray, ByteArray> {
        val keyStore = KeyStore.getInstance("AndroidKeyStore").apply { load(null) }
        val key = keyStore.getKey(KEY_ALIAS, null) as SecretKey
        
        val cipher = Cipher.getInstance("AES/GCM/NoPadding")
        cipher.init(Cipher.ENCRYPT_MODE, key)
        
        val encrypted = cipher.doFinal(data.toByteArray(Charsets.UTF_8))
        return Pair(encrypted, cipher.iv)
    }
    
    fun decrypt(encrypted: ByteArray, iv: ByteArray): String {
        val keyStore = KeyStore.getInstance("AndroidKeyStore").apply { load(null) }
        val key = keyStore.getKey(KEY_ALIAS, null) as SecretKey
        
        val cipher = Cipher.getInstance("AES/GCM/NoPadding")
        val spec = GCMParameterSpec(128, iv)
        cipher.init(Cipher.DECRYPT_MODE, key, spec)
        
        return String(cipher.doFinal(encrypted), Charsets.UTF_8)
    }
}
```

---

## ขั้นตอนที่ 854: Biometric Authentication

```kotlin
// implementation("androidx.biometric:biometric:1.2.x")

class BiometricAuthManager(private val activity: FragmentActivity) {
    
    private val biometricManager = BiometricManager.from(activity)
    
    fun canAuthenticate(): Boolean {
        return biometricManager.canAuthenticate(
            BiometricManager.Authenticators.BIOMETRIC_STRONG or
            BiometricManager.Authenticators.DEVICE_CREDENTIAL
        ) == BiometricManager.BIOMETRIC_SUCCESS
    }
    
    fun authenticate(
        onSuccess: () -> Unit,
        onError: (String) -> Unit,
        onFailed: () -> Unit
    ) {
        val promptInfo = BiometricPrompt.PromptInfo.Builder()
            .setTitle("ยืนยันตัวตน")
            .setSubtitle("ใช้ลายนิ้วมือหรือใบหน้าของคุณ")
            .setAllowedAuthenticators(
                BiometricManager.Authenticators.BIOMETRIC_STRONG or
                BiometricManager.Authenticators.DEVICE_CREDENTIAL
            )
            .build()
        
        val biometricPrompt = BiometricPrompt(
            activity,
            ContextCompat.getMainExecutor(activity),
            object : BiometricPrompt.AuthenticationCallback() {
                override fun onAuthenticationSucceeded(result: BiometricPrompt.AuthenticationResult) {
                    onSuccess()
                }
                
                override fun onAuthenticationError(errorCode: Int, errString: CharSequence) {
                    onError(errString.toString())
                }
                
                override fun onAuthenticationFailed() {
                    onFailed()
                }
            }
        )
        
        biometricPrompt.authenticate(promptInfo)
    }
}

// ViewModel Integration
@HiltViewModel
class SecureViewModel @Inject constructor(
    private val biometricManager: BiometricAuthManager
) : ViewModel() {
    
    private val _authState = MutableStateFlow<AuthState>(AuthState.NotAuthenticated)
    val authState: StateFlow<AuthState> = _authState.asStateFlow()
    
    fun authenticate() {
        biometricManager.authenticate(
            onSuccess = { _authState.value = AuthState.Authenticated },
            onError = { error -> _authState.value = AuthState.Error(error) },
            onFailed = { _authState.value = AuthState.Failed }
        )
    }
}

sealed class AuthState {
    object NotAuthenticated : AuthState()
    object Authenticated : AuthState()
    object Failed : AuthState()
    data class Error(val message: String) : AuthState()
}
```

---

## ขั้นตอนที่ 855: Secure Token Storage

```kotlin
// ============================================
// Token Manager - JWT storage และ management
// ============================================

class TokenManager(context: Context) {
    
    private val prefs = createEncryptedPrefs(context)
    
    companion object {
        private const val KEY_ACCESS_TOKEN = "access_token"
        private const val KEY_REFRESH_TOKEN = "refresh_token"
        private const val KEY_TOKEN_EXPIRY = "token_expiry"
    }
    
    fun saveTokens(accessToken: String, refreshToken: String, expiresIn: Long) {
        prefs.edit()
            .putString(KEY_ACCESS_TOKEN, accessToken)
            .putString(KEY_REFRESH_TOKEN, refreshToken)
            .putLong(KEY_TOKEN_EXPIRY, System.currentTimeMillis() + expiresIn * 1000)
            .apply()
    }
    
    fun getAccessToken(): String? = prefs.getString(KEY_ACCESS_TOKEN, null)
    
    fun getRefreshToken(): String? = prefs.getString(KEY_REFRESH_TOKEN, null)
    
    fun isTokenExpired(): Boolean {
        val expiry = prefs.getLong(KEY_TOKEN_EXPIRY, 0)
        return System.currentTimeMillis() > expiry - 60_000 // 1 minute buffer
    }
    
    fun clearTokens() {
        prefs.edit()
            .remove(KEY_ACCESS_TOKEN)
            .remove(KEY_REFRESH_TOKEN)
            .remove(KEY_TOKEN_EXPIRY)
            .apply()
    }
    
    fun hasValidToken(): Boolean {
        return getAccessToken() != null && !isTokenExpired()
    }
}

// ============================================
// Auth Interceptor กับ Token Refresh
// ============================================

class AuthInterceptor(
    private val tokenManager: TokenManager,
    private val refreshTokenApi: () -> Deferred<TokenResponse>
) : Interceptor {
    
    private val mutex = Mutex()
    
    override fun intercept(chain: Interceptor.Chain): Response {
        val originalRequest = chain.request()
        
        // ถ้าไม่ต้องการ auth header
        if (originalRequest.header("No-Auth") != null) {
            return chain.proceed(originalRequest)
        }
        
        val token = tokenManager.getAccessToken() ?: return chain.proceed(originalRequest)
        
        val request = originalRequest.newBuilder()
            .header("Authorization", "Bearer $token")
            .build()
        
        val response = chain.proceed(request)
        
        // 401 = token หมดอายุ
        if (response.code == 401) {
            response.close()
            
            // Refresh token
            val newToken = runBlocking {
                mutex.withLock {
                    if (!tokenManager.isTokenExpired()) {
                        tokenManager.getAccessToken()
                    } else {
                        try {
                            val refreshed = refreshTokenApi().await()
                            tokenManager.saveTokens(
                                refreshed.accessToken,
                                refreshed.refreshToken,
                                refreshed.expiresIn
                            )
                            refreshed.accessToken
                        } catch (e: Exception) {
                            tokenManager.clearTokens()
                            null
                        }
                    }
                }
            }
            
            if (newToken != null) {
                val newRequest = originalRequest.newBuilder()
                    .header("Authorization", "Bearer $newToken")
                    .build()
                return chain.proceed(newRequest)
            }
        }
        
        return response
    }
}
```

---

## แบบฝึกหัด Part 55

```kotlin
// แบบฝึกหัด: Implement Secure Note App

// TODO: สร้าง SecureNoteRepository ที่:
// 1. encrypt ก่อน save ลง Room ด้วย Android Keystore
// 2. ต้องผ่าน biometric auth ก่อนอ่าน
// 3. ใช้ EncryptedSharedPreferences เก็บ IV

@Entity(tableName = "secure_notes")
data class SecureNoteEntity(
    @PrimaryKey val id: String,
    val encryptedTitle: ByteArray,
    val encryptedContent: ByteArray,
    val iv: ByteArray,
    val createdAt: Long
)

// TODO:
// - encrypt(note: Note): SecureNoteEntity
// - decrypt(entity: SecureNoteEntity): Note  
// - Compose UI with biometric gate
```

---

*Part 55 จบแล้ว | ก่อนหน้า: [Part 54](../part54/README.md) | ถัดไป: [Part 56](../part56/README.md)*
