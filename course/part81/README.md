# Part 81: Advanced Security
## ขั้นตอนที่ 1501-1525

---

## ขั้นตอนที่ 1501: Security Threat Model

```
Android Security Layers:

1. Linux Kernel - process isolation, permissions
2. Application Sandbox - each app has own UID
3. SELinux - mandatory access control
4. Verified Boot - checks system integrity
5. Play Protect - scan apps for malware

Common Attack Vectors:
- Man-in-the-Middle (MITM)
- Reverse engineering
- Data extraction from storage
- Tapjacking / clickjacking
- SQL injection
- Intent hijacking
- Side-channel attacks

OWASP Mobile Top 10:
1. Improper Platform Usage
2. Insecure Data Storage
3. Insecure Communication
4. Insecure Authentication
5. Insufficient Cryptography
6. Insecure Authorization
7. Poor Code Quality
8. Code Tampering
9. Reverse Engineering
10. Extraneous Functionality
```

---

## ขั้นตอนที่ 1502: Root Detection

```kotlin
class RootDetector @Inject constructor() {
    
    fun isDeviceRooted(): Boolean {
        return checkSuBinary()
            || checkTestKeys()
            || checkDangerousProps()
            || checkRootPackages()
            || checkWritableSystemDirs()
    }
    
    private fun checkSuBinary(): Boolean {
        val paths = arrayOf(
            "/data/local/", "/data/local/bin/",
            "/data/local/xbin/", "/sbin/",
            "/su/bin/", "/system/bin/",
            "/system/bin/.ext/", "/system/sd/xbin/",
            "/system/usr/we-need-root/", "/system/xbin/"
        )
        return paths.any { File("${it}su").exists() }
    }
    
    private fun checkTestKeys(): Boolean {
        val buildTags = Build.TAGS
        return buildTags != null && buildTags.contains("test-keys")
    }
    
    private fun checkDangerousProps(): Boolean {
        val dangerousProps = mapOf(
            "ro.debuggable" to "1",
            "ro.secure" to "0"
        )
        
        dangerousProps.forEach { (prop, dangerousValue) ->
            val value = runCatching {
                val process = Runtime.getRuntime().exec(arrayOf("getprop", prop))
                process.inputStream.bufferedReader().readLine()
            }.getOrNull()
            
            if (value == dangerousValue) return true
        }
        return false
    }
    
    private fun checkRootPackages(): Boolean {
        val rootPackages = listOf(
            "com.noshufou.android.su",
            "com.noshufou.android.su.elite",
            "eu.chainfire.supersu",
            "com.koushikdutta.superuser",
            "com.thirdparty.superuser",
            "com.yellowes.su",
            "com.topjohnwu.magisk",
            "com.kingroot.kinguser"
        )
        
        return rootPackages.any { packageName ->
            try {
                // ApplicationContext is needed here
                false // simplified
            } catch (e: Exception) {
                false
            }
        }
    }
    
    private fun checkWritableSystemDirs(): Boolean {
        val writableFiles = arrayOf(
            File("/data/"), File("/etc/"),
            File("/proc/"), File("/sbin/"),
            File("/sys/"), File("/system/"),
            File("/system/bin"), File("/vendor/bin")
        )
        return writableFiles.any { it.canWrite() }
    }
}

// ใน Activity - block rooted devices
class SecureActivity : ComponentActivity() {
    
    @Inject lateinit var rootDetector: RootDetector
    
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        
        if (rootDetector.isDeviceRooted()) {
            showRootedDeviceWarning()
            return
        }
        
        // Continue normal setup
    }
    
    private fun showRootedDeviceWarning() {
        AlertDialog.Builder(this)
            .setTitle("อุปกรณ์ไม่ปลอดภัย")
            .setMessage("ตรวจพบว่าอุปกรณ์ของคุณถูก root ซึ่งอาจทำให้ข้อมูลไม่ปลอดภัย")
            .setPositiveButton("ออก") { _, _ -> finish() }
            .setCancelable(false)
            .show()
    }
}
```

---

## ขั้นตอนที่ 1503: Anti-Tamper & ProGuard

```kotlin
// ============================================
// Signature Verification - ตรวจสอบว่า APK ไม่ถูกแก้ไข
// ============================================

class AppIntegrityChecker @Inject constructor(
    @ApplicationContext private val context: Context
) {
    
    // Expected SHA-256 hash ของ signing certificate
    private val EXPECTED_CERT_HASH = "your_cert_sha256_hash_here"
    
    fun isAppTampered(): Boolean {
        return try {
            val packageInfo = if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.P) {
                context.packageManager.getPackageInfo(
                    context.packageName,
                    PackageManager.GET_SIGNING_CERTIFICATES
                )
            } else {
                @Suppress("DEPRECATION")
                context.packageManager.getPackageInfo(
                    context.packageName,
                    PackageManager.GET_SIGNATURES
                )
            }
            
            val actualHash = computeCertHash(packageInfo)
            actualHash != EXPECTED_CERT_HASH
            
        } catch (e: Exception) {
            true  // ถ้าหา certificate ไม่ได้ = tampered
        }
    }
    
    private fun computeCertHash(packageInfo: PackageInfo): String {
        val sigBytes = if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.P) {
            packageInfo.signingInfo.apkContentsSigners.first().toByteArray()
        } else {
            @Suppress("DEPRECATION")
            packageInfo.signatures.first().toByteArray()
        }
        
        val md = MessageDigest.getInstance("SHA-256")
        val hashBytes = md.digest(sigBytes)
        return hashBytes.joinToString("") { "%02x".format(it) }
    }
}

// ============================================
// ProGuard Rules (proguard-rules.pro)
// ============================================

/*
# Kotlin
-keep class kotlin.** { *; }
-keepclassmembers class **$WhenMappings { *; }

# Data classes used for JSON serialization
-keepclassmembers class com.myapp.data.** {
    @com.google.gson.annotations.SerializedName *;
}

# Room
-keep class * extends androidx.room.RoomDatabase { *; }
-keep @androidx.room.Entity class * { *; }
-keep @androidx.room.Dao interface * { *; }

# Retrofit
-keepattributes Signature
-keepattributes Exceptions
-keep class retrofit2.** { *; }

# Hilt
-keep class dagger.hilt.** { *; }
-keep class javax.inject.** { *; }

# Firebase
-keep class com.google.firebase.** { *; }

# Hide sensitive string in release builds
-assumenosideeffects class android.util.Log {
    public static boolean isLoggable(java.lang.String, int);
    public static int v(...);
    public static int d(...);
    public static int i(...);
}
*/
```

---

## ขั้นตอนที่ 1504: Screenshot Prevention & Secure Flag

```kotlin
// ============================================
// Prevent screenshot ในหน้าที่ sensitive
// ============================================

class SecureScreen : ComponentActivity() {
    
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        
        // Prevent screenshot and screen recording
        window.setFlags(
            WindowManager.LayoutParams.FLAG_SECURE,
            WindowManager.LayoutParams.FLAG_SECURE
        )
        
        setContent {
            SecureScreenContent()
        }
    }
}

// หรือใช้ Compose Modifier
@Composable
fun SecureContent(content: @Composable () -> Unit) {
    val activity = LocalContext.current as? Activity
    
    DisposableEffect(Unit) {
        activity?.window?.setFlags(
            WindowManager.LayoutParams.FLAG_SECURE,
            WindowManager.LayoutParams.FLAG_SECURE
        )
        
        onDispose {
            activity?.window?.clearFlags(WindowManager.LayoutParams.FLAG_SECURE)
        }
    }
    
    content()
}

// ใช้งาน
@Composable
fun PaymentScreen() {
    SecureContent {
        // UI นี้จะไม่สามารถ screenshot ได้
        Column {
            Text("หมายเลขบัตรเครดิต: xxxx-xxxx-xxxx-1234")
        }
    }
}
```

---

## ขั้นตอนที่ 1505: Network Security Advanced

```kotlin
// ============================================
// OkHttp Certificate Pinning
// ============================================

class NetworkSecurityConfig @Inject constructor() {
    
    fun createOkHttpClient(): OkHttpClient {
        val certificatePinner = CertificatePinner.Builder()
            // Production
            .add("api.myapp.com", "sha256/AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA=")
            // Backup pin (rotate certificates without breaking app)
            .add("api.myapp.com", "sha256/BBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBB=")
            .build()
        
        return OkHttpClient.Builder()
            .certificatePinner(certificatePinner)
            .connectTimeout(30, TimeUnit.SECONDS)
            .readTimeout(30, TimeUnit.SECONDS)
            .writeTimeout(30, TimeUnit.SECONDS)
            .addInterceptor(AuthInterceptor())
            .addInterceptor(LoggingInterceptor())
            .build()
    }
}

// ============================================
// JWT Token Management
// ============================================

class TokenManager @Inject constructor(
    private val encryptedPrefs: EncryptedSharedPreferences
) {
    
    private val KEY_ACCESS_TOKEN = "access_token"
    private val KEY_REFRESH_TOKEN = "refresh_token"
    private val KEY_EXPIRY = "token_expiry"
    
    fun saveTokens(accessToken: String, refreshToken: String, expirySeconds: Long) {
        val expiryTime = System.currentTimeMillis() + (expirySeconds * 1000)
        encryptedPrefs.edit()
            .putString(KEY_ACCESS_TOKEN, accessToken)
            .putString(KEY_REFRESH_TOKEN, refreshToken)
            .putLong(KEY_EXPIRY, expiryTime)
            .apply()
    }
    
    fun getAccessToken(): String? = encryptedPrefs.getString(KEY_ACCESS_TOKEN, null)
    
    fun getRefreshToken(): String? = encryptedPrefs.getString(KEY_REFRESH_TOKEN, null)
    
    fun isTokenExpired(): Boolean {
        val expiry = encryptedPrefs.getLong(KEY_EXPIRY, 0)
        return System.currentTimeMillis() >= expiry - 60_000  // refresh 1 min early
    }
    
    fun clearTokens() {
        encryptedPrefs.edit()
            .remove(KEY_ACCESS_TOKEN)
            .remove(KEY_REFRESH_TOKEN)
            .remove(KEY_EXPIRY)
            .apply()
    }
    
    fun parseTokenClaims(token: String): Map<String, Any>? {
        return try {
            // Decode JWT payload (base64url)
            val parts = token.split(".")
            if (parts.size != 3) return null
            
            val payload = Base64.decode(
                parts[1].replace('-', '+').replace('_', '/'),
                Base64.URL_SAFE or Base64.NO_PADDING
            )
            
            val json = String(payload)
            Gson().fromJson<Map<String, Any>>(json, object : TypeToken<Map<String, Any>>() {}.type)
        } catch (e: Exception) {
            null
        }
    }
}
```

---

## แบบฝึกหัด Part 81

```kotlin
// แบบฝึกหัด: Secure Storage Implementation

// สร้าง SecurePreferencesManager ที่:
// 1. ใช้ Android Keystore เพื่อ generate encryption key
// 2. Encrypt/Decrypt ด้วย AES-GCM
// 3. Store ใน SharedPreferences (cipher text)
// 4. Biometric authentication ก่อน access sensitive data
// 5. Key invalidation เมื่อ biometric changes

class SecurePreferencesManager @Inject constructor(
    @ApplicationContext private val context: Context
) {
    
    private val keyAlias = "myapp_secure_key"
    
    private fun getOrCreateKey(): SecretKey {
        val keyStore = KeyStore.getInstance("AndroidKeyStore")
        keyStore.load(null)
        
        if (!keyStore.containsAlias(keyAlias)) {
            val keyGenerator = KeyGenerator.getInstance(
                KeyProperties.KEY_ALGORITHM_AES,
                "AndroidKeyStore"
            )
            
            keyGenerator.init(
                KeyGenParameterSpec.Builder(
                    keyAlias,
                    KeyProperties.PURPOSE_ENCRYPT or KeyProperties.PURPOSE_DECRYPT
                )
                    .setBlockModes(KeyProperties.BLOCK_MODE_GCM)
                    .setEncryptionPaddings(KeyProperties.ENCRYPTION_PADDING_NONE)
                    .setUserAuthenticationRequired(true)  // require biometric
                    .setInvalidatedByBiometricEnrollment(true)
                    .build()
            )
            
            keyGenerator.generateKey()
        }
        
        return keyStore.getKey(keyAlias, null) as SecretKey
    }
    
    // TODO: implement encrypt/decrypt methods
    fun encrypt(plaintext: String): EncryptedData {
        TODO()
    }
    
    fun decrypt(data: EncryptedData): String {
        TODO()
    }
}

data class EncryptedData(val iv: ByteArray, val ciphertext: ByteArray)
```

---

*Part 81 จบแล้ว | ก่อนหน้า: [Part 80](../part80/README.md) | ถัดไป: [Part 82](../part82/README.md)*
