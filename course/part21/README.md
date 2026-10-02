# Part 21: แนะนำ Android SDK และ Android Studio
## ขั้นตอนที่ 501-520

---

## ขั้นตอนที่ 501: Android Development คืออะไร?

### ภาพรวม Android Ecosystem

```
Android Platform Stack:
┌────────────────────────────────────────┐
│           Applications                  │
│  (Calculator, Maps, Camera, Your App)  │
├────────────────────────────────────────┤
│         Application Framework          │
│  (Activity Manager, Window Manager,    │
│   Content Providers, View System)      │
├────────────────────────────────────────┤
│     Libraries + Android Runtime        │
│  (SQLite, WebKit, OpenGL, ART VM)     │
├────────────────────────────────────────┤
│          Linux Kernel                  │
│  (Drivers, Memory, Process Management)│
└────────────────────────────────────────┘
```

### Android SDK Components

```
Android SDK (Software Development Kit):
├── Build Tools (aapt, d8, zipalign)
├── Platform Tools (adb, fastboot)
├── Android Platforms (API levels 21-35)
├── System Images (for Emulator)
├── Support Libraries / AndroidX
└── Emulator
```

---

## ขั้นตอนที่ 502: สร้าง Android Project แรก

### ใน Android Studio

```
1. File > New > New Project
2. เลือก "Empty Activity" (Compose)
3. ตั้งค่า:
   - Name: HelloAndroid
   - Package name: com.example.helloandroid
   - Save location: เลือกโฟลเดอร์
   - Language: Kotlin
   - Minimum SDK: API 24 (Android 7.0) 
     → 95%+ ของอุปกรณ์รองรับ
4. คลิก Finish
5. รอ Gradle sync เสร็จ
```

---

## ขั้นตอนที่ 503: โครงสร้างโปรเจกต์ Android

```
HelloAndroid/
├── app/
│   ├── src/
│   │   ├── main/
│   │   │   ├── java/com/example/helloandroid/
│   │   │   │   └── MainActivity.kt          <- Activity หลัก
│   │   │   ├── res/
│   │   │   │   ├── drawable/               <- รูปภาพ, vectors
│   │   │   │   ├── layout/                 <- XML layouts (View system)
│   │   │   │   ├── mipmap-*/               <- App icons
│   │   │   │   ├── values/
│   │   │   │   │   ├── colors.xml          <- สีต่างๆ
│   │   │   │   │   ├── strings.xml         <- ข้อความ
│   │   │   │   │   └── themes.xml          <- Themes
│   │   │   │   └── xml/
│   │   │   └── AndroidManifest.xml         <- App configuration
│   │   ├── androidTest/                    <- Instrumented tests
│   │   └── test/                           <- Unit tests
│   └── build.gradle.kts                    <- App build config
├── gradle/
│   └── libs.versions.toml                  <- Version catalog
├── build.gradle.kts                        <- Project build config
└── settings.gradle.kts
```

---

## ขั้นตอนที่ 504: AndroidManifest.xml

```xml
<?xml version="1.0" encoding="utf-8"?>
<manifest xmlns:android="http://schemas.android.com/apk/res/android">

    <!-- Permissions ที่ต้องการ -->
    <uses-permission android:name="android.permission.INTERNET" />
    <uses-permission android:name="android.permission.ACCESS_NETWORK_STATE" />
    <!-- Camera permission (ต้องขอ runtime permission สำหรับ API 23+) -->
    <uses-permission android:name="android.permission.CAMERA" />
    
    <!-- Features ที่ต้องการ -->
    <uses-feature android:name="android.hardware.camera" android:required="false" />

    <application
        android:allowBackup="true"
        android:dataExtractionRules="@xml/data_extraction_rules"
        android:fullBackupContent="@xml/backup_rules"
        android:icon="@mipmap/ic_launcher"
        android:label="@string/app_name"
        android:roundIcon="@mipmap/ic_launcher_round"
        android:supportsRtl="true"
        android:theme="@style/Theme.HelloAndroid">
        
        <!-- Main Activity -->
        <activity
            android:name=".MainActivity"
            android:exported="true"  <!-- ต้องมีสำหรับ launcher -->
            android:label="@string/app_name"
            android:theme="@style/Theme.HelloAndroid">
            <intent-filter>
                <!-- App เริ่มต้นจาก Activity นี้ -->
                <action android:name="android.intent.action.MAIN" />
                <category android:name="android.intent.category.LAUNCHER" />
            </intent-filter>
        </activity>
        
        <!-- Activities อื่นๆ -->
        <activity android:name=".ProfileActivity" />
        <activity android:name=".SettingsActivity" />
        
        <!-- Service -->
        <service android:name=".MyBackgroundService" />
        
        <!-- BroadcastReceiver -->
        <receiver android:name=".NetworkChangeReceiver"
                  android:exported="false">
            <intent-filter>
                <action android:name="android.net.conn.CONNECTIVITY_CHANGE" />
            </intent-filter>
        </receiver>
        
        <!-- ContentProvider -->
        <provider
            android:name=".MyContentProvider"
            android:authorities="com.example.helloandroid.provider"
            android:exported="false" />
            
    </application>

</manifest>
```

---

## ขั้นตอนที่ 505: build.gradle.kts (App Level)

```kotlin
// app/build.gradle.kts

plugins {
    alias(libs.plugins.android.application)
    alias(libs.plugins.kotlin.android)
    alias(libs.plugins.kotlin.compose)
    
    // Kotlin Symbol Processing (KSP) สำหรับ Room, Hilt
    alias(libs.plugins.ksp)
    
    // Hilt
    alias(libs.plugins.hilt)
}

android {
    namespace = "com.example.helloandroid"
    compileSdk = 35  // ใช้ SDK ล่าสุดในการ compile
    
    defaultConfig {
        applicationId = "com.example.helloandroid"
        minSdk = 24        // Android 7.0 (รองรับ ~95% ของอุปกรณ์)
        targetSdk = 35     // ออกแบบสำหรับ API 35
        versionCode = 1    // Internal version (ต้องเพิ่มทุก release)
        versionName = "1.0.0"  // User-visible version
        
        testInstrumentationRunner = "androidx.test.runner.AndroidJUnitRunner"
        
        // ตั้งค่า BuildConfig fields
        buildConfigField("String", "API_BASE_URL", "\"https://api.example.com\"")
        buildConfigField("Boolean", "ENABLE_LOGGING", "true")
        
        // Room schema export location
        javaCompileOptions {
            annotationProcessorOptions {
                arguments["room.schemaLocation"] = "$projectDir/schemas"
            }
        }
    }
    
    // Build Types
    buildTypes {
        debug {
            applicationIdSuffix = ".debug"
            versionNameSuffix = "-DEBUG"
            isDebuggable = true
            buildConfigField("Boolean", "ENABLE_LOGGING", "true")
        }
        
        release {
            isMinifyEnabled = true   // ProGuard/R8
            isShrinkResources = true
            proguardFiles(
                getDefaultProguardFile("proguard-android-optimize.txt"),
                "proguard-rules.pro"
            )
            buildConfigField("Boolean", "ENABLE_LOGGING", "false")
            signingConfig = signingConfigs.getByName("release")
        }
    }
    
    // Product Flavors (สำหรับ app หลายเวอร์ชัน)
    flavorDimensions += "env"
    productFlavors {
        create("staging") {
            dimension = "env"
            applicationIdSuffix = ".staging"
            buildConfigField("String", "API_BASE_URL", "\"https://staging.api.example.com\"")
        }
        create("production") {
            dimension = "env"
            buildConfigField("String", "API_BASE_URL", "\"https://api.example.com\"")
        }
    }
    
    compileOptions {
        sourceCompatibility = JavaVersion.VERSION_17
        targetCompatibility = JavaVersion.VERSION_17
        isCoreLibraryDesugaringEnabled = true  // Java 8+ APIs บน API < 26
    }
    
    kotlinOptions {
        jvmTarget = "17"
    }
    
    buildFeatures {
        compose = true
        buildConfig = true
    }
    
    packaging {
        resources {
            excludes += "/META-INF/{AL2.0,LGPL2.1}"
        }
    }
}

dependencies {
    // Core
    implementation(libs.androidx.core.ktx)
    implementation(libs.androidx.lifecycle.runtime.ktx)
    
    // Compose
    implementation(platform(libs.androidx.compose.bom))
    implementation(libs.androidx.ui)
    implementation(libs.androidx.ui.graphics)
    implementation(libs.androidx.ui.tooling.preview)
    implementation(libs.androidx.material3)
    implementation(libs.androidx.activity.compose)
    
    // ViewModel + Navigation
    implementation(libs.androidx.lifecycle.viewmodel.compose)
    implementation(libs.androidx.navigation.compose)
    
    // Coroutines
    implementation(libs.kotlinx.coroutines.android)
    
    // Room
    implementation(libs.androidx.room.runtime)
    implementation(libs.androidx.room.ktx)
    ksp(libs.androidx.room.compiler)
    
    // Retrofit
    implementation(libs.retrofit)
    implementation(libs.retrofit.converter.gson)
    implementation(libs.okhttp3.logging.interceptor)
    
    // Hilt
    implementation(libs.hilt.android)
    ksp(libs.hilt.compiler)
    implementation(libs.androidx.hilt.navigation.compose)
    
    // Coil (Image loading)
    implementation(libs.coil.compose)
    
    // Desugaring
    coreLibraryDesugaring(libs.android.tools.desugar)
    
    // Testing
    testImplementation(libs.junit)
    testImplementation(libs.kotlinx.coroutines.test)
    androidTestImplementation(libs.androidx.junit)
    androidTestImplementation(libs.androidx.espresso.core)
    androidTestImplementation(platform(libs.androidx.compose.bom))
    androidTestImplementation(libs.androidx.ui.test.junit4)
    debugImplementation(libs.androidx.ui.tooling)
    debugImplementation(libs.androidx.ui.test.manifest)
}
```

---

## ขั้นตอนที่ 506: Version Catalog (libs.versions.toml)

```toml
# gradle/libs.versions.toml

[versions]
agp = "8.5.0"
kotlin = "2.0.0"
ksp = "2.0.0-1.0.21"
coreKtx = "1.13.1"
junit = "4.13.2"
junitVersion = "1.2.1"
espressoCore = "3.6.1"
lifecycleRuntimeKtx = "2.8.2"
activityCompose = "1.9.0"
composeBom = "2024.06.00"
navigationCompose = "2.7.7"
room = "2.6.1"
retrofit = "2.11.0"
okhttp = "4.12.0"
hilt = "2.51.1"
hiltNavigationCompose = "1.2.0"
coil = "2.6.0"
coroutines = "1.8.1"
desugar = "2.0.4"

[libraries]
# AndroidX Core
androidx-core-ktx = { group = "androidx.core", name = "core-ktx", version.ref = "coreKtx" }
androidx-lifecycle-runtime-ktx = { group = "androidx.lifecycle", name = "lifecycle-runtime-ktx", version.ref = "lifecycleRuntimeKtx" }
androidx-lifecycle-viewmodel-compose = { group = "androidx.lifecycle", name = "lifecycle-viewmodel-compose", version.ref = "lifecycleRuntimeKtx" }
androidx-activity-compose = { group = "androidx.activity", name = "activity-compose", version.ref = "activityCompose" }

# Compose
androidx-compose-bom = { group = "androidx.compose", name = "compose-bom", version.ref = "composeBom" }
androidx-ui = { group = "androidx.compose.ui", name = "ui" }
androidx-ui-graphics = { group = "androidx.compose.ui", name = "ui-graphics" }
androidx-ui-tooling = { group = "androidx.compose.ui", name = "ui-tooling" }
androidx-ui-tooling-preview = { group = "androidx.compose.ui", name = "ui-tooling-preview" }
androidx-ui-test-manifest = { group = "androidx.compose.ui", name = "ui-test-manifest" }
androidx-ui-test-junit4 = { group = "androidx.compose.ui", name = "ui-test-junit4" }
androidx-material3 = { group = "androidx.compose.material3", name = "material3" }

# Navigation
androidx-navigation-compose = { group = "androidx.navigation", name = "navigation-compose", version.ref = "navigationCompose" }

# Room
androidx-room-runtime = { group = "androidx.room", name = "room-runtime", version.ref = "room" }
androidx-room-ktx = { group = "androidx.room", name = "room-ktx", version.ref = "room" }
androidx-room-compiler = { group = "androidx.room", name = "room-compiler", version.ref = "room" }

# Retrofit + OkHttp
retrofit = { group = "com.squareup.retrofit2", name = "retrofit", version.ref = "retrofit" }
retrofit-converter-gson = { group = "com.squareup.retrofit2", name = "converter-gson", version.ref = "retrofit" }
okhttp3-logging-interceptor = { group = "com.squareup.okhttp3", name = "logging-interceptor", version.ref = "okhttp" }

# Hilt
hilt-android = { group = "com.google.dagger", name = "hilt-android", version.ref = "hilt" }
hilt-compiler = { group = "com.google.dagger", name = "hilt-android-compiler", version.ref = "hilt" }
androidx-hilt-navigation-compose = { group = "androidx.hilt", name = "hilt-navigation-compose", version.ref = "hiltNavigationCompose" }

# Coil
coil-compose = { group = "io.coil-kt", name = "coil-compose", version.ref = "coil" }

# Coroutines
kotlinx-coroutines-android = { group = "org.jetbrains.kotlinx", name = "kotlinx-coroutines-android", version.ref = "coroutines" }
kotlinx-coroutines-test = { group = "org.jetbrains.kotlinx", name = "kotlinx-coroutines-test", version.ref = "coroutines" }

# Desugaring
android-tools-desugar = { group = "com.android.tools", name = "desugar_jdk_libs", version.ref = "desugar" }

# Testing
junit = { group = "junit", name = "junit", version.ref = "junit" }
androidx-junit = { group = "androidx.test.ext", name = "junit", version.ref = "junitVersion" }
androidx-espresso-core = { group = "androidx.test.espresso", name = "espresso-core", version.ref = "espressoCore" }

[plugins]
android-application = { id = "com.android.application", version.ref = "agp" }
android-library = { id = "com.android.library", version.ref = "agp" }
kotlin-android = { id = "org.jetbrains.kotlin.android", version.ref = "kotlin" }
kotlin-compose = { id = "org.jetbrains.kotlin.plugin.compose", version.ref = "kotlin" }
ksp = { id = "com.google.devtools.ksp", version.ref = "ksp" }
hilt = { id = "com.google.dagger.hilt.android", version.ref = "hilt" }
```

---

## ขั้นตอนที่ 507: MainActivity พื้นฐาน

```kotlin
// app/src/main/java/com/example/helloandroid/MainActivity.kt

package com.example.helloandroid

import android.os.Bundle
import androidx.activity.ComponentActivity
import androidx.activity.compose.setContent
import androidx.activity.enableEdgeToEdge
import androidx.compose.foundation.layout.*
import androidx.compose.material3.*
import androidx.compose.runtime.*
import androidx.compose.ui.Alignment
import androidx.compose.ui.Modifier
import androidx.compose.ui.text.style.TextAlign
import androidx.compose.ui.tooling.preview.Preview
import androidx.compose.ui.unit.dp
import com.example.helloandroid.ui.theme.HelloAndroidTheme

class MainActivity : ComponentActivity() {
    
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        
        // Edge-to-edge display (ใช้พื้นที่เต็มหน้าจอ)
        enableEdgeToEdge()
        
        // Set Compose UI
        setContent {
            HelloAndroidTheme {
                // Scaffold จัดการ system bars, FAB, etc.
                Scaffold(modifier = Modifier.fillMaxSize()) { innerPadding ->
                    HelloScreen(
                        modifier = Modifier.padding(innerPadding)
                    )
                }
            }
        }
    }
}

@Composable
fun HelloScreen(modifier: Modifier = Modifier) {
    // State ใน Compose
    var count by remember { mutableIntStateOf(0) }
    var name by remember { mutableStateOf("") }
    
    Column(
        modifier = modifier
            .fillMaxSize()
            .padding(16.dp),
        horizontalAlignment = Alignment.CenterHorizontally,
        verticalArrangement = Arrangement.Center
    ) {
        // Title
        Text(
            text = "สวัสดี Android!",
            style = MaterialTheme.typography.headlineLarge
        )
        
        Spacer(modifier = Modifier.height(24.dp))
        
        // Counter
        Text(
            text = "นับ: $count",
            style = MaterialTheme.typography.headlineMedium
        )
        
        Spacer(modifier = Modifier.height(16.dp))
        
        Row(horizontalArrangement = Arrangement.spacedBy(8.dp)) {
            Button(onClick = { count-- }) {
                Text("-")
            }
            Button(onClick = { count++ }) {
                Text("+")
            }
            OutlinedButton(onClick = { count = 0 }) {
                Text("Reset")
            }
        }
        
        Spacer(modifier = Modifier.height(24.dp))
        
        // Text Input
        OutlinedTextField(
            value = name,
            onValueChange = { name = it },
            label = { Text("ชื่อของคุณ") },
            placeholder = { Text("กรอกชื่อ...") }
        )
        
        Spacer(modifier = Modifier.height(8.dp))
        
        if (name.isNotBlank()) {
            Text(
                text = "สวัสดี, $name! 👋",
                style = MaterialTheme.typography.bodyLarge,
                textAlign = TextAlign.Center
            )
        }
    }
}

@Preview(showBackground = true, showSystemUi = true)
@Composable
fun HelloScreenPreview() {
    HelloAndroidTheme {
        Scaffold { innerPadding ->
            HelloScreen(modifier = Modifier.padding(innerPadding))
        }
    }
}
```

---

## ขั้นตอนที่ 508: Android App Lifecycle

```
Activity Lifecycle:
                    ┌─────────────────┐
                    │   onCreate()     │ ← App เริ่มต้น / หลัง killed
                    └────────┬────────┘
                             │
                    ┌────────▼────────┐
                    │   onStart()     │ ← Activity กำลังจะแสดง
                    └────────┬────────┘
                             │
                    ┌────────▼────────┐
           ┌──────► │   onResume()    │ ← User interact ได้แล้ว
           │        └────────┬────────┘
           │                 │
           │         User กลับมา
           │                 │
           │        ┌────────▼────────┐
           │        │   onPause()     │ ← Activity บางส่วนถูกบัง
           │        └────────┬────────┘
           │                 │
           │        ┌────────▼────────┐
           │        │   onStop()      │ ← Activity ไม่แสดงอีกแล้ว
           │        └────────┬────────┘
           │                 │
           │    ┌────────────┴────────────┐
           │    │                         │
           │  กลับมา              ┌────────▼────────┐
           │    │                 │  onDestroy()    │ ← Activity ถูกทำลาย
           │    │                 └─────────────────┘
           │    │
           │  ┌─▼───────────────┐
           └──│   onRestart()   │
              └─────────────────┘
```

```kotlin
class LifecycleExampleActivity : ComponentActivity() {
    
    private val TAG = "LifecycleDemo"
    
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        // Activity สร้างใหม่
        // ดึง savedInstanceState ถ้ามี
        Log.d(TAG, "onCreate: savedState = $savedInstanceState")
    }
    
    override fun onStart() {
        super.onStart()
        // Activity กำลังจะแสดง
        // เริ่ม listeners, resume animations
        Log.d(TAG, "onStart")
    }
    
    override fun onResume() {
        super.onResume()
        // Activity foreground, user interact ได้
        // เริ่ม camera, GPS, real-time updates
        Log.d(TAG, "onResume")
    }
    
    override fun onPause() {
        super.onPause()
        // Activity ยังเห็นอยู่แต่ focus ไปที่อื่น
        // หยุด animation, ปล่อย camera
        // ทำงานเร็วๆ ที่นี่ (block UI ไม่ได้)
        Log.d(TAG, "onPause")
    }
    
    override fun onStop() {
        super.onStop()
        // Activity ไม่แสดงอีกแล้ว
        // บันทึก data, หยุด heavy work
        Log.d(TAG, "onStop")
    }
    
    override fun onRestart() {
        super.onRestart()
        // จาก onStop กลับมา
        Log.d(TAG, "onRestart")
    }
    
    override fun onDestroy() {
        super.onDestroy()
        // Activity ถูก destroy
        // Clean up resources
        Log.d(TAG, "onDestroy")
    }
    
    override fun onSaveInstanceState(outState: Bundle) {
        super.onSaveInstanceState(outState)
        // บันทึก state ก่อน Activity ถูก kill
        outState.putInt("count", 5)
        outState.putString("text", "some data")
        Log.d(TAG, "onSaveInstanceState")
    }
}
```

---

## ขั้นตอนที่ 509: Android Emulator Setup

```bash
# สร้าง AVD (Android Virtual Device) ผ่าน Android Studio:
# Tools > Device Manager > Create Virtual Device

# หรือผ่าน Command Line:
# ค้นหา system images ที่มี
sdkmanager --list | grep "system-images"

# ดาวน์โหลด system image
sdkmanager "system-images;android-34;google_apis;x86_64"

# สร้าง AVD
avdmanager create avd \
    -n "Pixel7_API34" \
    -k "system-images;android-34;google_apis;x86_64" \
    -d "pixel_7"

# รัน Emulator
emulator -avd Pixel7_API34

# ADB Commands ที่ใช้บ่อย
adb devices                          # list connected devices
adb install app-debug.apk            # install APK
adb uninstall com.example.helloandroid  # uninstall
adb shell                            # shell access
adb logcat                           # view logs
adb logcat -s "MainActivity"        # filter by tag
adb push local_file /sdcard/         # copy file to device
adb pull /sdcard/file local_dir      # copy file from device
adb shell pm list packages          # list installed packages
adb shell am start -n com.example/.MainActivity  # start activity
adb shell input tap 500 1000         # simulate tap
adb shell screencap /sdcard/screen.png  # screenshot
```

---

## ขั้นตอนที่ 510: Resource Files

### strings.xml

```xml
<!-- res/values/strings.xml -->
<resources>
    <string name="app_name">Hello Android</string>
    
    <!-- Basic strings -->
    <string name="btn_submit">ยืนยัน</string>
    <string name="btn_cancel">ยกเลิก</string>
    <string name="msg_loading">กำลังโหลด...</string>
    
    <!-- String with formatting -->
    <string name="welcome_user">สวัสดี, %s!</string>
    <string name="item_count">มี %d รายการ</string>
    
    <!-- Plurals -->
    <plurals name="items_count">
        <item quantity="one">%d รายการ</item>
        <item quantity="other">%d รายการ</item>
    </plurals>
    
    <!-- String array -->
    <string-array name="thai_months">
        <item>มกราคม</item>
        <item>กุมภาพันธ์</item>
        <item>มีนาคม</item>
        <item>เมษายน</item>
        <item>พฤษภาคม</item>
        <item>มิถุนายน</item>
        <item>กรกฎาคม</item>
        <item>สิงหาคม</item>
        <item>กันยายน</item>
        <item>ตุลาคม</item>
        <item>พฤศจิกายน</item>
        <item>ธันวาคม</item>
    </string-array>
</resources>
```

### colors.xml

```xml
<!-- res/values/colors.xml -->
<resources>
    <!-- Brand Colors -->
    <color name="primary">#1976D2</color>
    <color name="primary_dark">#0D47A1</color>
    <color name="primary_light">#BBDEFB</color>
    <color name="secondary">#FF5722</color>
    <color name="secondary_dark">#BF360C</color>
    
    <!-- Semantic Colors -->
    <color name="success">#4CAF50</color>
    <color name="warning">#FF9800</color>
    <color name="error">#F44336</color>
    <color name="info">#2196F3</color>
    
    <!-- Neutral Colors -->
    <color name="black">#000000</color>
    <color name="white">#FFFFFF</color>
    <color name="gray_100">#F5F5F5</color>
    <color name="gray_300">#E0E0E0</color>
    <color name="gray_500">#9E9E9E</color>
    <color name="gray_700">#616161</color>
    <color name="gray_900">#212121</color>
    
    <!-- Background Colors -->
    <color name="background">#FAFAFA</color>
    <color name="surface">#FFFFFF</color>
    <color name="on_primary">#FFFFFF</color>
    <color name="on_background">#212121</color>
</resources>
```

### themes.xml

```xml
<!-- res/values/themes.xml -->
<resources xmlns:tools="http://schemas.android.com/tools">
    <style name="Theme.HelloAndroid" parent="Theme.Material3.DayNight.NoActionBar">
        <!-- Customize Material 3 theme -->
        <item name="colorPrimary">@color/primary</item>
        <item name="colorPrimaryContainer">@color/primary_light</item>
        <item name="colorSecondary">@color/secondary</item>
        
        <!-- Typography -->
        <item name="android:fontFamily">@font/noto_sans_thai</item>
        
        <!-- Status bar -->
        <item name="android:statusBarColor">@android:color/transparent</item>
    </style>
</resources>
```

---

## สรุป Part 21

ใน Part นี้เราได้เรียนรู้:
1. ภาพรวม Android Platform Stack
2. การสร้าง Android Project ใน Android Studio
3. โครงสร้างโปรเจกต์และไฟล์สำคัญ
4. AndroidManifest.xml - การตั้งค่า App
5. build.gradle.kts - การ configure build
6. Version Catalog (libs.versions.toml)
7. MainActivity พื้นฐานด้วย Jetpack Compose
8. Activity Lifecycle
9. ADB Commands
10. Resource Files (strings, colors, themes)

---

*Part 21 จบแล้ว | ก่อนหน้า: [Part 20](../part20/README.md) | ถัดไป: [Part 22](../part22/README.md)*
