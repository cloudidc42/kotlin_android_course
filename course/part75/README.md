# Part 75: Custom Gradle Plugins & Build System
## ขั้นตอนที่ 1351-1375

---

## ขั้นตอนที่ 1351: Gradle Build System

```
Gradle Build System ใน Android:

Project Structure:
├── build.gradle.kts (root)
├── settings.gradle.kts
├── gradle/
│   ├── libs.versions.toml  (Version Catalog)
│   └── wrapper/
├── app/
│   └── build.gradle.kts
└── core/
    └── build.gradle.kts

Version Catalog (libs.versions.toml):
- Central place สำหรับ dependency versions
- Type-safe access ใน build.gradle.kts

Convention Plugins:
- Shared build config ระหว่าง modules
- ลด duplication ใน build files
```

---

## ขั้นตอนที่ 1352: Version Catalog

```toml
# gradle/libs.versions.toml
[versions]
kotlin = "2.x"
agp = "8.x"
compose = "1.x"
composeBom = "2024.x"
hilt = "2.x"
room = "2.x"
retrofit = "2.x"
coroutines = "1.x"
lifecycle = "2.x"

[libraries]
# Kotlin
kotlin-stdlib = { module = "org.jetbrains.kotlin:kotlin-stdlib", version.ref = "kotlin" }
kotlinx-coroutines-android = { module = "org.jetbrains.kotlinx:kotlinx-coroutines-android", version.ref = "coroutines" }
kotlinx-coroutines-test = { module = "org.jetbrains.kotlinx:kotlinx-coroutines-test", version.ref = "coroutines" }

# AndroidX Core
androidx-core-ktx = { module = "androidx.core:core-ktx", version = "1.x" }
androidx-appcompat = { module = "androidx.appcompat:appcompat", version = "1.x" }

# Compose
compose-bom = { module = "androidx.compose:compose-bom", version.ref = "composeBom" }
compose-ui = { module = "androidx.compose.ui:ui" }
compose-ui-tooling = { module = "androidx.compose.ui:ui-tooling" }
compose-material3 = { module = "androidx.compose.material3:material3" }
compose-runtime = { module = "androidx.compose.runtime:runtime" }

# Lifecycle
lifecycle-viewmodel-ktx = { module = "androidx.lifecycle:lifecycle-viewmodel-ktx", version.ref = "lifecycle" }
lifecycle-runtime-compose = { module = "androidx.lifecycle:lifecycle-runtime-compose", version.ref = "lifecycle" }

# Hilt
hilt-android = { module = "com.google.dagger:hilt-android", version.ref = "hilt" }
hilt-compiler = { module = "com.google.dagger:hilt-android-compiler", version.ref = "hilt" }

# Room
room-runtime = { module = "androidx.room:room-runtime", version.ref = "room" }
room-ktx = { module = "androidx.room:room-ktx", version.ref = "room" }
room-compiler = { module = "androidx.room:room-compiler", version.ref = "room" }

# Retrofit
retrofit-core = { module = "com.squareup.retrofit2:retrofit", version.ref = "retrofit" }
retrofit-gson = { module = "com.squareup.retrofit2:converter-gson", version.ref = "retrofit" }

[bundles]
# Group dependencies
compose = ["compose-ui", "compose-material3", "compose-runtime"]
room = ["room-runtime", "room-ktx"]

[plugins]
android-application = { id = "com.android.application", version.ref = "agp" }
android-library = { id = "com.android.library", version.ref = "agp" }
kotlin-android = { id = "org.jetbrains.kotlin.android", version.ref = "kotlin" }
kotlin-kapt = { id = "org.jetbrains.kotlin.kapt", version.ref = "kotlin" }
hilt = { id = "com.google.dagger.hilt.android", version.ref = "hilt" }
ksp = { id = "com.google.devtools.ksp", version = "2.x" }
```

---

## ขั้นตอนที่ 1353: Convention Plugins

```kotlin
// build-logic/convention/src/main/kotlin/
// AndroidLibraryConventionPlugin.kt

class AndroidLibraryConventionPlugin : Plugin<Project> {
    
    override fun apply(target: Project) {
        with(target) {
            with(pluginManager) {
                apply("com.android.library")
                apply("org.jetbrains.kotlin.android")
            }
            
            extensions.configure<LibraryExtension> {
                configureKotlinAndroid(this)
                defaultConfig.targetSdk = 35
                defaultConfig.testInstrumentationRunner = "androidx.test.runner.AndroidJUnitRunner"
                testOptions.animationsDisabled = true
            }
            
            // Add common dependencies
            val libs = extensions.getByType<VersionCatalogsExtension>().named("libs")
            dependencies {
                "implementation"(libs.findLibrary("androidx.core.ktx").get())
                "implementation"(libs.findLibrary("kotlinx.coroutines.android").get())
                "testImplementation"(libs.findLibrary("junit").get())
                "testImplementation"(libs.findLibrary("kotlinx.coroutines.test").get())
            }
        }
    }
}

// AndroidApplicationConventionPlugin.kt
class AndroidApplicationConventionPlugin : Plugin<Project> {
    override fun apply(target: Project) {
        with(target) {
            with(pluginManager) {
                apply("com.android.application")
                apply("org.jetbrains.kotlin.android")
            }
            
            extensions.configure<ApplicationExtension> {
                configureKotlinAndroid(this)
                defaultConfig.targetSdk = 35
                defaultConfig.versionCode = 1
                defaultConfig.versionName = "1.0.0"
                
                buildTypes {
                    release {
                        isMinifyEnabled = true
                        proguardFiles(
                            getDefaultProguardFile("proguard-android-optimize.txt"),
                            "proguard-rules.pro"
                        )
                    }
                }
            }
        }
    }
}

// Common config function
internal fun Project.configureKotlinAndroid(commonExtension: CommonExtension<*, *, *, *, *, *>) {
    commonExtension.apply {
        compileSdk = 35
        
        defaultConfig {
            minSdk = 26
        }
        
        compileOptions {
            sourceCompatibility = JavaVersion.VERSION_17
            targetCompatibility = JavaVersion.VERSION_17
            isCoreLibraryDesugaringEnabled = true
        }
        
        kotlinOptions {
            jvmTarget = "17"
            freeCompilerArgs = freeCompilerArgs + listOf(
                "-opt-in=kotlin.RequiresOptIn",
                "-opt-in=kotlinx.coroutines.ExperimentalCoroutinesApi",
                "-opt-in=androidx.compose.material3.ExperimentalMaterial3Api"
            )
        }
    }
    
    dependencies {
        "coreLibraryDesugaring"(
            libs.findLibrary("android.desugarJdkLibs").get()
        )
    }
}
```

---

## ขั้นตอนที่ 1354: Convention Plugin Registration

```kotlin
// build-logic/convention/build.gradle.kts

plugins {
    `kotlin-dsl`
}

group = "com.myapp.buildlogic"

java {
    sourceCompatibility = JavaVersion.VERSION_17
    targetCompatibility = JavaVersion.VERSION_17
}

dependencies {
    compileOnly(libs.android.gradlePlugin)
    compileOnly(libs.kotlin.gradlePlugin)
    compileOnly(libs.ksp.gradlePlugin)
}

// Register plugins
gradlePlugin {
    plugins {
        register("androidApplication") {
            id = "myapp.android.application"
            implementationClass = "AndroidApplicationConventionPlugin"
        }
        register("androidLibrary") {
            id = "myapp.android.library"
            implementationClass = "AndroidLibraryConventionPlugin"
        }
        register("androidCompose") {
            id = "myapp.android.compose"
            implementationClass = "AndroidComposeConventionPlugin"
        }
        register("androidHilt") {
            id = "myapp.android.hilt"
            implementationClass = "AndroidHiltConventionPlugin"
        }
        register("androidRoom") {
            id = "myapp.android.room"
            implementationClass = "AndroidRoomConventionPlugin"
        }
    }
}
```

---

## ขั้นตอนที่ 1355: Using Convention Plugins

```kotlin
// settings.gradle.kts - เพิ่ม build-logic
pluginManagement {
    includeBuild("build-logic")
    repositories {
        google()
        mavenCentral()
        gradlePluginPortal()
    }
}

// app/build.gradle.kts - ใช้ convention plugins
plugins {
    alias(libs.plugins.myapp.android.application)  // Convention plugin
    alias(libs.plugins.myapp.android.compose)
    alias(libs.plugins.myapp.android.hilt)
}

android {
    namespace = "com.myapp"
    
    defaultConfig {
        applicationId = "com.myapp"
    }
}

dependencies {
    implementation(project(":core:common"))
    implementation(project(":core:ui"))
    implementation(project(":feature:home"))
}

// core/data/build.gradle.kts
plugins {
    alias(libs.plugins.myapp.android.library)
    alias(libs.plugins.myapp.android.hilt)
    alias(libs.plugins.myapp.android.room)
}

android {
    namespace = "com.myapp.core.data"
}

dependencies {
    implementation(project(":core:model"))
    implementation(project(":core:network"))
    implementation(libs.kotlinx.coroutines.android)
}
```

---

## แบบฝึกหัด Part 75

```kotlin
// แบบฝึกหัด: สร้าง Convention Plugin ใหม่

// 1. สร้าง AndroidTestingConventionPlugin ที่:
//    - เพิ่ม JUnit, MockK, Turbine, Coroutines Test
//    - Config testOptions
//    - เปิด useTestStorageService

// 2. สร้าง AndroidFeatureConventionPlugin ที่:
//    - รวม AndroidLibrary + Compose + Hilt
//    - เพิ่ม Navigation Compose dependency
//    - เพิ่ม hiltViewModel, collectAsStateWithLifecycle

class AndroidTestingConventionPlugin : Plugin<Project> {
    override fun apply(target: Project) {
        with(target) {
            // TODO: implement
        }
    }
}

class AndroidFeatureConventionPlugin : Plugin<Project> {
    override fun apply(target: Project) {
        with(target) {
            pluginManager.apply("myapp.android.library")
            pluginManager.apply("myapp.android.compose")
            pluginManager.apply("myapp.android.hilt")
            
            // TODO: add feature-specific dependencies
        }
    }
}
```

---

*Part 75 จบแล้ว | ก่อนหน้า: [Part 74](../part74/README.md) | ถัดไป: [Part 76](../part76/README.md)*
