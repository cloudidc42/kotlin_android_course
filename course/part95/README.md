# Part 95: CI/CD & Release Engineering
## ขั้นตอนที่ 1851-1875

---

## ขั้นตอนที่ 1851: GitHub Actions สำหรับ Android

```yaml
# .github/workflows/android-ci.yml
name: Android CI

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Set up JDK 17
        uses: actions/setup-java@v4
        with:
          java-version: '17'
          distribution: 'temurin'
          cache: gradle
      
      - name: Grant execute permission
        run: chmod +x gradlew
      
      - name: Run unit tests
        run: ./gradlew test --parallel
      
      - name: Run lint
        run: ./gradlew lint
      
      - name: Upload test results
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: test-results
          path: '**/build/test-results/**/*.xml'
      
      - name: Upload lint results
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: lint-results
          path: '**/build/reports/lint-results*.html'
  
  build:
    needs: test
    runs-on: ubuntu-latest
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Set up JDK 17
        uses: actions/setup-java@v4
        with:
          java-version: '17'
          distribution: 'temurin'
          cache: gradle
      
      - name: Decode keystore
        env:
          KEYSTORE_BASE64: ${{ secrets.KEYSTORE_BASE64 }}
        run: |
          echo $KEYSTORE_BASE64 | base64 --decode > app/keystore.jks
      
      - name: Build release APK
        env:
          KEYSTORE_PASSWORD: ${{ secrets.KEYSTORE_PASSWORD }}
          KEY_ALIAS: ${{ secrets.KEY_ALIAS }}
          KEY_PASSWORD: ${{ secrets.KEY_PASSWORD }}
        run: |
          ./gradlew assembleRelease \
            -Pandroid.injected.signing.store.file=keystore.jks \
            -Pandroid.injected.signing.store.password=$KEYSTORE_PASSWORD \
            -Pandroid.injected.signing.key.alias=$KEY_ALIAS \
            -Pandroid.injected.signing.key.password=$KEY_PASSWORD
      
      - name: Upload APK
        uses: actions/upload-artifact@v4
        with:
          name: release-apk
          path: app/build/outputs/apk/release/*.apk

  deploy:
    needs: build
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Download APK
        uses: actions/download-artifact@v4
        with:
          name: release-apk
      
      - name: Upload to Firebase App Distribution
        uses: wzieba/Firebase-Distribution-Github-Action@v1
        with:
          appId: ${{ secrets.FIREBASE_APP_ID }}
          serviceCredentialsFileContent: ${{ secrets.FIREBASE_SERVICE_CREDENTIALS }}
          groups: testers
          file: app-release.apk
          releaseNotes: |
            ${{ github.event.head_commit.message }}
```

---

## ขั้นตอนที่ 1852: Fastlane Setup

```ruby
# fastlane/Fastfile

default_platform(:android)

platform :android do
  
  desc "Run all tests"
  lane :test do
    gradle(
      task: "test",
      flags: "--parallel --build-cache"
    )
  end
  
  desc "Build release APK"
  lane :build_release do
    gradle(
      task: "assemble",
      build_type: "Release",
      properties: {
        "android.injected.signing.store.file" => ENV["KEYSTORE_PATH"],
        "android.injected.signing.store.password" => ENV["KEYSTORE_PASSWORD"],
        "android.injected.signing.key.alias" => ENV["KEY_ALIAS"],
        "android.injected.signing.key.password" => ENV["KEY_PASSWORD"],
      }
    )
  end
  
  desc "Build release AAB"
  lane :build_bundle do
    gradle(
      task: "bundle",
      build_type: "Release",
      properties: {
        "android.injected.signing.store.file" => ENV["KEYSTORE_PATH"],
        "android.injected.signing.store.password" => ENV["KEYSTORE_PASSWORD"],
        "android.injected.signing.key.alias" => ENV["KEY_ALIAS"],
        "android.injected.signing.key.password" => ENV["KEY_PASSWORD"],
      }
    )
  end
  
  desc "Deploy to Firebase App Distribution"
  lane :deploy_firebase do
    build_release
    
    firebase_app_distribution(
      app: ENV["FIREBASE_APP_ID"],
      service_credentials_file: "firebase-service-account.json",
      groups: "internal-testers",
      release_notes: last_git_commit[:message]
    )
  end
  
  desc "Deploy to Play Store (Internal testing)"
  lane :deploy_internal do
    build_bundle
    
    upload_to_play_store(
      track: "internal",
      aab: lane_context[SharedValues::GRADLE_AAB_OUTPUT_PATH],
      skip_upload_apk: true,
      skip_upload_metadata: true,
      skip_upload_images: true,
      skip_upload_screenshots: true
    )
  end
  
  desc "Deploy to Play Store (Production) with 10% rollout"
  lane :deploy_production do |options|
    build_bundle
    
    upload_to_play_store(
      track: "production",
      rollout: options[:rollout] || "0.1",
      aab: lane_context[SharedValues::GRADLE_AAB_OUTPUT_PATH],
      skip_upload_apk: true
    )
  end
  
  desc "Bump version code"
  lane :bump_version do
    path = "../app/build.gradle.kts"
    re = /versionCode = (\d+)/
    
    content = File.read(path)
    current = content.match(re)[1].to_i
    new_version = current + 1
    
    File.write(path, content.sub(re, "versionCode = #{new_version}"))
    
    git_commit(path: path, message: "Bump version code to #{new_version}")
  end
end
```

---

## ขั้นตอนที่ 1853: Versioning Strategy

```kotlin
// ============================================
// Semantic Versioning for Android
// ============================================

// build.gradle.kts (app)
android {
    defaultConfig {
        // versionCode: auto-increment integer (Play Store)
        versionCode = System.getenv("VERSION_CODE")?.toInt() ?: 1
        
        // versionName: semantic version (shown to users)
        versionName = "2.1.0"
    }
}

// ============================================
// Version from Git tags
// ============================================

// build.gradle.kts (project root)
fun getVersionCode(): Int {
    return try {
        val process = Runtime.getRuntime().exec("git rev-list --count HEAD")
        process.inputStream.bufferedReader().readLine().trim().toInt()
    } catch (e: Exception) {
        1
    }
}

fun getVersionName(): String {
    return try {
        val process = Runtime.getRuntime().exec("git describe --tags --abbrev=0")
        process.inputStream.bufferedReader().readLine().trim().removePrefix("v")
    } catch (e: Exception) {
        "1.0.0"
    }
}

android {
    defaultConfig {
        versionCode = getVersionCode()
        versionName = getVersionName()
    }
}
```

---

## ขั้นตอนที่ 1854: Play Store Release Management

```
Play Store Track Strategy:

Internal Testing (testers ที่ invite เท่านั้น)
↓ ผ่าน QA
Alpha (closed testing group)
↓ ผ่าน Beta testers
Beta (open testing)
↓ เสถียรพอ
Production (สาธารณะ)

Staged Rollout:
- Production: เริ่มที่ 1-5%
- Monitor crash rate, ANR rate 24 ชั่วโมง
- ถ้าดี → เพิ่มเป็น 10%, 25%, 50%, 100%
- ถ้าแย่ → halt rollout, investigate, fix

Key Metrics to Monitor:
- Crash-free sessions rate (target: >99.5%)
- ANR rate (target: <0.24%)
- Rating (target: ≥4.0)
- Uninstall rate

Pre-launch Report:
- ทดสอบอัตโนมัติบน Firebase Test Lab
- Robo test กับ devices หลายรุ่น
- Accessibility check
- Security check
```

---

## ขั้นตอนที่ 1855: App Signing Setup

```kotlin
// ============================================
// App Signing Configuration
// ============================================

// Create keystore (ทำครั้งเดียว)
// keytool -genkeypair -v -keystore myapp.jks -alias myapp -keyalg RSA -keysize 2048 -validity 10000

// build.gradle.kts (app)
android {
    signingConfigs {
        create("release") {
            // Option 1: Environment variables (CI/CD)
            storeFile = System.getenv("KEYSTORE_PATH")?.let { file(it) }
            storePassword = System.getenv("KEYSTORE_PASSWORD")
            keyAlias = System.getenv("KEY_ALIAS")
            keyPassword = System.getenv("KEY_PASSWORD")
            
            // Option 2: local.properties (local dev only - never commit!)
            // val keystoreProps = Properties().apply {
            //     load(rootProject.file("keystore.properties").inputStream())
            // }
            // storeFile = file(keystoreProps.getProperty("storeFile"))
            // storePassword = keystoreProps.getProperty("storePassword")
            // keyAlias = keystoreProps.getProperty("keyAlias")
            // keyPassword = keystoreProps.getProperty("keyPassword")
        }
    }
    
    buildTypes {
        release {
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

// .gitignore
/*
# Keystore (NEVER commit these)
*.jks
*.keystore
keystore.properties
google-services.json  # Contains API keys
*/

// GitHub Secrets setup:
// KEYSTORE_BASE64: base64 encoded keystore file
//   → base64 myapp.jks | pbcopy
// KEYSTORE_PASSWORD: storePassword
// KEY_ALIAS: key alias
// KEY_PASSWORD: keyPassword
```

---

*Part 95 จบแล้ว | ก่อนหน้า: [Part 94](../part94/README.md) | ถัดไป: [Part 96](../part96/README.md)*
