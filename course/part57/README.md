# Part 57: CI/CD and Release
## ขั้นตอนที่ 901-925

---

## ขั้นตอนที่ 901: CI/CD คืออะไร?

```
CI/CD Pipeline สำหรับ Android:

┌─────────┐    ┌──────────┐    ┌─────────┐    ┌──────────┐
│  Push   │───▶│   CI    │───▶│  Build  │───▶│ Release  │
│  Code   │    │(Test +  │    │  APK/   │    │  Play    │
│         │    │  Lint)  │    │  AAB    │    │  Store   │
└─────────┘    └──────────┘    └─────────┘    └──────────┘

เครื่องมือ CI/CD ยอดนิยม:
- GitHub Actions ✅ (ฟรีสำหรับ open source)
- Bitrise (Android-focused)
- CircleCI
- GitLab CI
- Fastlane (automation tool)
```

---

## ขั้นตอนที่ 902: GitHub Actions

```yaml
# .github/workflows/android.yml
name: Android CI

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

jobs:
  test:
    name: Run Tests
    runs-on: ubuntu-latest
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Set up JDK 17
        uses: actions/setup-java@v4
        with:
          java-version: '17'
          distribution: 'temurin'
          cache: gradle
      
      - name: Grant execute permission for gradlew
        run: chmod +x gradlew
      
      - name: Run Lint
        run: ./gradlew lint
      
      - name: Run Unit Tests
        run: ./gradlew testDebugUnitTest
      
      - name: Upload Test Results
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: test-results
          path: app/build/reports/tests/
      
      - name: Upload Lint Results
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: lint-results
          path: app/build/reports/lint-results-debug.html

  build:
    name: Build APK
    runs-on: ubuntu-latest
    needs: test
    if: github.ref == 'refs/heads/main'
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Set up JDK 17
        uses: actions/setup-java@v4
        with:
          java-version: '17'
          distribution: 'temurin'
          cache: gradle
      
      - name: Decode Keystore
        run: |
          echo "${{ secrets.KEYSTORE_BASE64 }}" | base64 -d > release.jks
      
      - name: Build Release APK
        run: |
          ./gradlew assembleRelease \
            -Pandroid.injected.signing.store.file=release.jks \
            -Pandroid.injected.signing.store.password=${{ secrets.KEYSTORE_PASSWORD }} \
            -Pandroid.injected.signing.key.alias=${{ secrets.KEY_ALIAS }} \
            -Pandroid.injected.signing.key.password=${{ secrets.KEY_PASSWORD }}
      
      - name: Upload APK
        uses: actions/upload-artifact@v4
        with:
          name: release-apk
          path: app/build/outputs/apk/release/*.apk
```

---

## ขั้นตอนที่ 903: App Signing

```kotlin
// build.gradle.kts (app)
android {
    signingConfigs {
        create("release") {
            // อ่านจาก environment variables (ปลอดภัยกว่า hardcode)
            storeFile = System.getenv("KEYSTORE_PATH")?.let { file(it) }
                ?: file("../keystore/release.jks")
            storePassword = System.getenv("KEYSTORE_PASSWORD") ?: properties["keystorePassword"]?.toString()
            keyAlias = System.getenv("KEY_ALIAS") ?: properties["keyAlias"]?.toString()
            keyPassword = System.getenv("KEY_PASSWORD") ?: properties["keyPassword"]?.toString()
        }
    }
    
    buildTypes {
        getByName("release") {
            signingConfig = signingConfigs.getByName("release")
            isMinifyEnabled = true
            isShrinkResources = true
            proguardFiles(
                getDefaultProguardFile("proguard-android-optimize.txt"),
                "proguard-rules.pro"
            )
        }
    }
}

// local.properties (ไม่ commit เข้า git!)
// keystorePassword=mypassword
// keyAlias=mykey
// keyPassword=mykeypassword
```

```
# สร้าง Keystore ด้วย keytool:
keytool -genkey -v \
  -keystore release.jks \
  -alias my-app-key \
  -keyalg RSA \
  -keysize 2048 \
  -validity 10000

# ดู certificate info:
keytool -list -v -keystore release.jks

# SHA-1 fingerprint (สำหรับ Firebase, Google Maps):
keytool -list -v -keystore release.jks -alias my-app-key | grep SHA1
```

---

## ขั้นตอนที่ 904: ProGuard / R8

```
# proguard-rules.pro

# Keep data classes (prevent field name obfuscation)
-keepclassmembers class com.example.app.data.model.** {
    <fields>;
}

# Retrofit
-keepattributes Signature
-keepattributes *Annotation*
-keep class retrofit2.** { *; }
-keep interface retrofit2.** { *; }

# Gson
-keepattributes Signature
-keepattributes *Annotation*
-keep class com.google.gson.** { *; }
-keep class * implements com.google.gson.TypeAdapterFactory
-keep class * implements com.google.gson.JsonSerializer
-keep class * implements com.google.gson.JsonDeserializer

# Kotlinx Serialization
-keepattributes *Annotation*, InnerClasses
-dontnote kotlinx.serialization.AnnotationsKt
-keepclassmembers class kotlinx.serialization.json.** {
    *** Companion;
}

# Hilt
-keep class dagger.hilt.** { *; }
-keep class javax.inject.** { *; }

# OkHttp
-dontwarn okhttp3.**
-dontwarn okio.**

# Room
-keep class * extends androidx.room.RoomDatabase
-keep @androidx.room.Database class *

# Coroutines
-keepnames class kotlinx.coroutines.internal.MainDispatcherFactory {}
-keepnames class kotlinx.coroutines.CoroutineExceptionHandler {}

# ป้องกัน crash จาก missing classes
-dontwarn java.lang.invoke.StringConcatFactory
```

---

## ขั้นตอนที่ 905: Fastlane

```ruby
# Fastfile

default_platform(:android)

platform :android do
  
  desc "Run all tests"
  lane :test do
    gradle(task: "testDebugUnitTest")
    gradle(task: "connectedDebugAndroidTest")
  end
  
  desc "Build and upload to Play Store Internal Testing"
  lane :internal do
    # Increment version
    increment_version_code
    
    # Build AAB
    gradle(
      task: "bundle",
      build_type: "Release",
      properties: {
        "android.injected.signing.store.file" => ENV["KEYSTORE_PATH"],
        "android.injected.signing.store.password" => ENV["KEYSTORE_PASSWORD"],
        "android.injected.signing.key.alias" => ENV["KEY_ALIAS"],
        "android.injected.signing.key.password" => ENV["KEY_PASSWORD"]
      }
    )
    
    # Upload to Play Store
    upload_to_play_store(
      track: 'internal',
      release_status: 'draft',
      aab: 'app/build/outputs/bundle/release/app-release.aab'
    )
    
    # Notify Slack
    slack(
      message: "New internal build uploaded!",
      channel: "#android-releases"
    )
  end
  
  desc "Deploy to production"
  lane :production do
    # Confirm
    UI.confirm("Are you sure you want to release to production?")
    
    build_and_sign
    
    upload_to_play_store(
      track: 'production',
      rollout: '0.1'  # 10% rollout
    )
  end
  
  private_lane :build_and_sign do
    gradle(
      task: "bundle",
      build_type: "Release"
    )
  end
end
```

---

## ขั้นตอนที่ 906: Play Store Release

```kotlin
// App Bundle vs APK:
// AAB (Android App Bundle) - แนะนำ:
//   - Google Play แปลงเป็น APK ที่เหมาะกับแต่ละ device
//   - ขนาดเล็กกว่า 15-50%
//   - จำเป็นสำหรับ new apps (บังคับตั้งแต่ August 2021)
//
// APK:
//   - Single file
//   - ง่ายสำหรับ direct install
//   - ใช้ได้กับ sideloading

// วางแผน Release Track:
// Internal Testing → Alpha → Beta → Production

// Gradle สำหรับ version management:
// build.gradle.kts

android {
    defaultConfig {
        versionCode = getVersionCode()  // auto-increment ใน CI
        versionName = "1.0.0"
    }
}

fun getVersionCode(): Int {
    // ใน CI/CD ใช้ build number
    val envCode = System.getenv("BUILD_NUMBER")?.toIntOrNull()
    if (envCode != null) return envCode
    
    // local: อ่านจาก file
    val versionFile = file("version.properties")
    val props = java.util.Properties()
    if (versionFile.exists()) {
        props.load(versionFile.inputStream())
    }
    return props.getProperty("versionCode", "1").toInt()
}
```

---

## แบบฝึกหัด Part 57

```yaml
# TODO: สร้าง GitHub Actions workflow ที่:
# 1. Run tests ทุก PR
# 2. Build debug APK และ upload เป็น artifact
# 3. Run lint check
# 4. Comment ผลลัพธ์ใน PR
# 5. Build release AAB เฉพาะ merge เข้า main

# Hint: ใช้ actions/upload-artifact สำหรับ upload
# และ peter-evans/create-or-update-comment สำหรับ comment

# .github/workflows/pr-check.yml
name: PR Check
on:
  pull_request:
    branches: [main]
jobs:
  check:
    runs-on: ubuntu-latest
    steps:
      # TODO: complete the workflow steps
      - uses: actions/checkout@v4
      # ... เพิ่ม steps ที่เหลือ
```

---

*Part 57 จบแล้ว | ก่อนหน้า: [Part 56](../part56/README.md) | ถัดไป: [Part 58](../part58/README.md)*
