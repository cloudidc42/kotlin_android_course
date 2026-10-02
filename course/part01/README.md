# Part 01: แนะนำ Kotlin และการติดตั้ง
## ขั้นตอนที่ 1-10

---

## ขั้นตอนที่ 1: Kotlin คืออะไร?

Kotlin เป็นภาษาโปรแกรมที่พัฒนาโดย JetBrains ในปี 2011 และ Google ประกาศให้เป็นภาษาหลักสำหรับการพัฒนา Android ในปี 2019

### จุดเด่นของ Kotlin

```kotlin
// 1. Concise - โค้ดกระชับกว่า Java มาก
// Java แบบเดิม:
// public class Person {
//     private String name;
//     private int age;
//     public Person(String name, int age) {
//         this.name = name;
//         this.age = age;
//     }
//     // getters, setters, toString, equals, hashCode...
// }

// Kotlin แบบใหม่ - บรรทัดเดียว!
data class Person(val name: String, val age: Int)

// 2. Null Safety - ป้องกัน NullPointerException
val name: String = "สมชาย"        // ไม่สามารถเป็น null ได้
val nickname: String? = null       // สามารถเป็น null ได้ (ต้องใส่ ?)

// 3. Interoperable กับ Java 100%
// สามารถใช้ Kotlin และ Java ร่วมกันในโปรเจกต์เดียวกันได้

// 4. สนับสนุน Functional Programming
val numbers = listOf(1, 2, 3, 4, 5)
val evenNumbers = numbers.filter { it % 2 == 0 }  // [2, 4]
val doubled = numbers.map { it * 2 }               // [2, 4, 6, 8, 10]

// 5. Coroutines - สำหรับ Asynchronous Programming
// (จะเรียนใน Part 14-15)
```

### เปรียบเทียบ Kotlin vs Java

| คุณสมบัติ | Kotlin | Java |
|-----------|--------|------|
| Null Safety | ✅ Built-in | ❌ ต้องใช้ Optional |
| Data Classes | ✅ 1 บรรทัด | ❌ หลายสิบบรรทัด |
| Smart Casts | ✅ อัตโนมัติ | ❌ ต้อง cast เอง |
| Coroutines | ✅ Native | ❌ ต้องใช้ Library |
| Extension Functions | ✅ มี | ❌ ไม่มี |
| String Templates | ✅ มี | ❌ ต้องใช้ + |

---

## ขั้นตอนที่ 2: การติดตั้ง IntelliJ IDEA (สำหรับ Kotlin ทั่วไป)

### สำหรับ macOS

```bash
# วิธีที่ 1: ดาวน์โหลดจากเว็บไซต์
# https://www.jetbrains.com/idea/download/
# เลือก Community Edition (ฟรี)

# วิธีที่ 2: ใช้ Homebrew
brew install --cask intellij-idea-ce

# ตรวจสอบการติดตั้ง Java (ต้องมี JDK 11 หรือสูงกว่า)
java -version
# ถ้าไม่มี ติดตั้ง:
brew install openjdk@17
```

### สำหรับ Windows

```powershell
# วิธีที่ 1: ดาวน์โหลดจาก https://www.jetbrains.com/idea/download/

# วิธีที่ 2: ใช้ Chocolatey
choco install intellijidea-community

# หรือ Scoop
scoop install intellij-idea

# ตรวจสอบ Java
java -version
# ถ้าไม่มี ดาวน์โหลด JDK จาก:
# https://adoptium.net/
```

### สำหรับ Ubuntu/Linux

```bash
# วิธีที่ 1: Ubuntu Software Center
# ค้นหา "IntelliJ IDEA"

# วิธีที่ 2: Snap
sudo snap install intellij-idea-community --classic

# วิธีที่ 3: ดาวน์โหลด tar.gz
wget https://download.jetbrains.com/idea/ideaIC-2024.1.tar.gz
tar -xzf ideaIC-2024.1.tar.gz
cd idea-IC-*/bin
./idea.sh

# ติดตั้ง Java
sudo apt update
sudo apt install openjdk-17-jdk
java -version
```

---

## ขั้นตอนที่ 3: การติดตั้ง Android Studio (สำหรับ Android Development)

Android Studio คือ IDE หลักสำหรับการพัฒนา Android ซึ่งรวม IntelliJ IDEA มาด้วย

### ดาวน์โหลดและติดตั้ง

```bash
# macOS - ใช้ Homebrew
brew install --cask android-studio

# Windows - ดาวน์โหลดจาก
# https://developer.android.com/studio

# Linux - Ubuntu
sudo snap install android-studio --classic
# หรือดาวน์โหลด tar.gz จาก official website

# ตรวจสอบ JAVA_HOME (สำคัญมาก!)
echo $JAVA_HOME
# ถ้าว่าง ให้ตั้งค่า:

# macOS/Linux - เพิ่มใน ~/.bashrc หรือ ~/.zshrc:
export JAVA_HOME=$(/usr/libexec/java_home -v 17)  # macOS
export JAVA_HOME=/usr/lib/jvm/java-17-openjdk-amd64  # Ubuntu
export PATH=$JAVA_HOME/bin:$PATH

# Windows - ตั้งใน System Environment Variables
# JAVA_HOME = C:\Program Files\Eclipse Adoptium\jdk-17.x.x.x-hotspot
```

### การตั้งค่า Android Studio ครั้งแรก

```
1. เปิด Android Studio
2. คลิก "Next" ผ่าน Setup Wizard
3. เลือก "Standard" Installation Type
4. เลือก UI Theme (Dark/Light)
5. ยืนยันการติดตั้ง SDK Components
   - Android SDK Platform 34
   - Android SDK Build-Tools
   - Android Emulator
   - Android SDK Platform-Tools
6. คลิก "Finish"
```

### ตรวจสอบการติดตั้ง

```bash
# ตรวจสอบ Android SDK
ls ~/Library/Android/sdk/  # macOS
ls ~/Android/Sdk/           # Linux
# Windows: C:\Users\YourName\AppData\Local\Android\Sdk

# ตรวจสอบ adb (Android Debug Bridge)
adb version
# Android Debug Bridge version X.X.X

# เพิ่ม Platform-Tools ใน PATH
# macOS/Linux - เพิ่มใน ~/.bashrc หรือ ~/.zshrc:
export ANDROID_HOME=$HOME/Library/Android/sdk  # macOS
export ANDROID_HOME=$HOME/Android/Sdk           # Linux
export PATH=$PATH:$ANDROID_HOME/emulator
export PATH=$PATH:$ANDROID_HOME/platform-tools
export PATH=$PATH:$ANDROID_HOME/tools
export PATH=$PATH:$ANDROID_HOME/tools/bin
```

---

## ขั้นตอนที่ 4: สร้างโปรเจกต์ Kotlin แรก

### ใช้ IntelliJ IDEA

```
1. เปิด IntelliJ IDEA
2. คลิก "New Project"
3. เลือก "Kotlin" ทางซ้าย
4. ตั้งค่า:
   - Name: HelloKotlin
   - Location: เลือกโฟลเดอร์ที่ต้องการ
   - Build System: Gradle (แนะนำ)
   - JDK: 17
   - Kotlin DSL: ✅
5. คลิก "Create"
```

### โครงสร้างโปรเจกต์

```
HelloKotlin/
├── build.gradle.kts          <- Build configuration
├── settings.gradle.kts       <- Project settings
├── gradle/
│   └── wrapper/
│       └── gradle-wrapper.properties
├── src/
│   ├── main/
│   │   └── kotlin/
│   │       └── Main.kt       <- ไฟล์หลัก
│   └── test/
│       └── kotlin/
│           └── MainTest.kt   <- ไฟล์ทดสอบ
└── .gitignore
```

### ไฟล์ build.gradle.kts

```kotlin
// build.gradle.kts
plugins {
    kotlin("jvm") version "2.0.0"
}

group = "com.example"
version = "1.0-SNAPSHOT"

repositories {
    mavenCentral()
}

dependencies {
    testImplementation(kotlin("test"))
}

tasks.test {
    useJUnitPlatform()
}

kotlin {
    jvmToolchain(17)
}
```

---

## ขั้นตอนที่ 5: Hello World! โปรแกรมแรกใน Kotlin

### สร้างไฟล์ Main.kt

```kotlin
// src/main/kotlin/Main.kt

fun main() {
    println("Hello, World!")
    println("สวัสดี, Kotlin!")
    
    // แสดงข้อมูลเพิ่มเติม
    val name = "นักเรียน Kotlin"
    val version = 2.0
    
    println("ยินดีต้อนรับ $name")
    println("Kotlin version: $version")
}
```

### รันโปรแกรม

```bash
# วิธีที่ 1: ใน IntelliJ IDEA
# คลิกปุ่ม ▶ (Run) ข้าง fun main()

# วิธีที่ 2: ใช้ Gradle
./gradlew run  # macOS/Linux
gradlew run    # Windows

# วิธีที่ 3: ใช้ Command Line
# ติดตั้ง Kotlin compiler ก่อน
kotlinc Main.kt -include-runtime -d main.jar
java -jar main.jar
```

### ผลลัพธ์

```
Hello, World!
สวัสดี, Kotlin!
ยินดีต้อนรับ นักเรียน Kotlin
Kotlin version: 2.0
```

---

## ขั้นตอนที่ 6: ทำความเข้าใจโครงสร้างโค้ด Kotlin

```kotlin
// ไฟล์ Main.kt

// Package declaration (ไม่บังคับแต่แนะนำ)
package com.example.hello

// Import statements
import java.util.Date
import kotlin.math.sqrt

// Top-level function (ฟังก์ชันที่ระดับบนสุด ไม่ต้องอยู่ใน Class)
fun main() {
    // Entry point ของโปรแกรม
    
    // การประกาศตัวแปรแบบ val (immutable - เปลี่ยนแปลงค่าไม่ได้)
    val greeting = "Hello"
    
    // การประกาศตัวแปรแบบ var (mutable - เปลี่ยนแปลงค่าได้)
    var count = 0
    count = 1  // ✅ ได้
    // greeting = "Hi"  // ❌ Error! val เปลี่ยนค่าไม่ได้
    
    // String Template
    println("$greeting, World! Count: $count")
    
    // String Template แบบ expression
    println("2 + 2 = ${2 + 2}")
    
    // เรียกใช้ฟังก์ชันอื่น
    greet("สมชาย")
    calculateArea(5.0, 3.0)
}

// ฟังก์ชันอื่นๆ
fun greet(name: String) {
    println("สวัสดี, $name!")
}

fun calculateArea(width: Double, height: Double): Double {
    val area = width * height
    println("พื้นที่ = $area")
    return area
}

// Object (Singleton) - คล้าย static ใน Java
object MathUtils {
    fun square(n: Int): Int = n * n
    fun cube(n: Int): Int = n * n * n
}
```

---

## ขั้นตอนที่ 7: การใช้ Kotlin REPL และ Online Playground

### Kotlin REPL (Read-Eval-Print Loop)

```bash
# เปิด Kotlin REPL ใน Terminal
kotlinc

# ทดลองพิมพ์:
>>> val x = 10
>>> val y = 20
>>> println(x + y)
30
>>> "Hello".uppercase()
res2: kotlin.String = HELLO
>>> listOf(1,2,3).sum()
res3: kotlin.Int = 6
>>> :quit  // ออกจาก REPL
```

### ใน IntelliJ IDEA

```
Tools > Kotlin > Kotlin REPL
```

### Kotlin Playground (Online)

```
เปิด Browser ไปที่: https://play.kotlinlang.org/

ลองพิมพ์โค้ดนี้:
```

```kotlin
// Kotlin Playground Example
fun main() {
    val fruits = listOf("มะม่วง", "กล้วย", "ส้ม", "แอปเปิ้ล")
    
    println("ผลไม้ทั้งหมด:")
    fruits.forEachIndexed { index, fruit ->
        println("${index + 1}. $fruit")
    }
    
    val longNames = fruits.filter { it.length > 3 }
    println("\nผลไม้ที่ชื่อยาวกว่า 3 ตัวอักษร: $longNames")
    
    val upperFruits = fruits.map { it.uppercase() }
    println("ตัวพิมพ์ใหญ่: $upperFruits")
}
```

---

## ขั้นตอนที่ 8: ทำความเข้าใจ Kotlin Compilation Process

```
┌─────────────────────────────────────────────────┐
│              Kotlin Source Code (.kt)            │
└─────────────────────┬───────────────────────────┘
                      │
                      ▼
┌─────────────────────────────────────────────────┐
│            Kotlin Compiler (kotlinc)             │
└─────────────────────┬───────────────────────────┘
                      │
          ┌───────────┼────────────┐
          ▼           ▼            ▼
   ┌──────────┐ ┌──────────┐ ┌──────────┐
   │ JVM       │ │JavaScript│ │  Native  │
   │Bytecode   │ │  Code    │ │ Binary   │
   │  (.class) │ │  (.js)   │ │          │
   └──────────┘ └──────────┘ └──────────┘
        │
        ▼
   ┌──────────┐
   │   JVM    │
   │(Java VM) │
   └──────────┘
```

### ตัวอย่างการ Compile และ Run

```bash
# ขั้นตอนการ compile โดยตรง
# 1. สร้างไฟล์ Hello.kt
cat > Hello.kt << 'EOF'
fun main() {
    println("Hello from compiled Kotlin!")
    
    for (i in 1..5) {
        println("Step $i")
    }
}
EOF

# 2. Compile
kotlinc Hello.kt -include-runtime -d hello.jar

# 3. Run
java -jar hello.jar

# ผลลัพธ์:
# Hello from compiled Kotlin!
# Step 1
# Step 2
# Step 3
# Step 4
# Step 5

# ดู bytecode
javap -c HelloKt.class
```

---

## ขั้นตอนที่ 9: การตั้งค่า Code Style และ Formatter

### EditorConfig

```ini
# .editorconfig ในโฟลเดอร์โปรเจกต์
root = true

[*]
charset = utf-8
end_of_line = lf
indent_style = space
indent_size = 4
trim_trailing_whitespace = true
insert_final_newline = true

[*.{kt,kts}]
indent_size = 4
max_line_length = 120
```

### Ktlint (Kotlin Linter)

```kotlin
// build.gradle.kts - เพิ่ม ktlint plugin
plugins {
    kotlin("jvm") version "2.0.0"
    id("org.jlleitschuh.gradle.ktlint") version "12.1.0"
}

// ตั้งค่า ktlint
ktlint {
    version.set("1.2.1")
    verbose.set(true)
    android.set(false)
    outputToConsole.set(true)
    ignoreFailures.set(false)
    enableExperimentalRules.set(true)
    filter {
        exclude("**/generated/**")
        include("**/kotlin/**")
    }
}
```

```bash
# รัน ktlint check
./gradlew ktlintCheck

# แก้ไขอัตโนมัติ
./gradlew ktlintFormat
```

### Detekt (Static Analysis)

```kotlin
// build.gradle.kts
plugins {
    id("io.gitlab.arturbosch.detekt") version "1.23.6"
}

detekt {
    config.setFrom("$projectDir/config/detekt.yml")
    buildUponDefaultConfig = true
    autoCorrect = true
}
```

---

## ขั้นตอนที่ 10: สรุปและแบบฝึกหัด Part 01

### สรุปสิ่งที่เรียนใน Part 01

1. **Kotlin คืออะไร** - ภาษาสมัยใหม่จาก JetBrains, ใช้กับ Android เป็นหลัก
2. **การติดตั้ง** - IntelliJ IDEA สำหรับ Kotlin ทั่วไป, Android Studio สำหรับ Android
3. **โปรแกรมแรก** - Hello World, ทำความเข้าใจโครงสร้างโค้ด
4. **Compilation** - Kotlin compile ไปเป็น JVM bytecode, JS, หรือ Native
5. **เครื่องมือ** - REPL, Playground, ktlint, detekt

### แบบฝึกหัด

```kotlin
// แบบฝึกหัดที่ 1: แก้โค้ดนี้ให้ถูกต้อง
fun main() {
    val message = "Hello"
    // TODO: เปลี่ยน message เป็น "Hi" - hint: ต้องเปลี่ยน val เป็นอะไร?
    message = "Hi"
    println(message)
}

// แบบฝึกหัดที่ 2: เติมโค้ดให้ครบ
fun printInfo(name: String, age: Int) {
    // TODO: พิมพ์ "ชื่อ: [name], อายุ: [age] ปี"
}

// แบบฝึกหัดที่ 3: สร้างฟังก์ชันคำนวณ BMI
// BMI = น้ำหนัก(kg) / (ส่วนสูง(m) * ส่วนสูง(m))
fun calculateBMI(weightKg: Double, heightM: Double): Double {
    // TODO: คืนค่า BMI
    return 0.0
}

fun main() {
    printInfo("สมชาย", 25)
    val bmi = calculateBMI(70.0, 1.75)
    println("BMI = $bmi")
}
```

### เฉลย

```kotlin
// เฉลยแบบฝึกหัดที่ 1:
fun main() {
    var message = "Hello"  // เปลี่ยน val เป็น var
    message = "Hi"
    println(message)  // Hi
}

// เฉลยแบบฝึกหัดที่ 2:
fun printInfo(name: String, age: Int) {
    println("ชื่อ: $name, อายุ: $age ปี")
}

// เฉลยแบบฝึกหัดที่ 3:
fun calculateBMI(weightKg: Double, heightM: Double): Double {
    return weightKg / (heightM * heightM)
}

fun main() {
    printInfo("สมชาย", 25)
    val bmi = calculateBMI(70.0, 1.75)
    println("BMI = %.2f".format(bmi))  // BMI = 22.86
    
    // ตีความ BMI
    val interpretation = when {
        bmi < 18.5 -> "น้ำหนักน้อยเกินไป"
        bmi < 25.0 -> "น้ำหนักปกติ"
        bmi < 30.0 -> "น้ำหนักเกิน"
        else -> "อ้วน"
    }
    println("ผล: $interpretation")
}
```

---

## Workshop: สร้าง Calculator พื้นฐาน

```kotlin
// Calculator.kt - โปรแกรมคิดเลขพื้นฐาน

fun main() {
    println("=== Kotlin Calculator ===")
    println("เลือกการคำนวณ:")
    println("1. บวก (+)")
    println("2. ลบ (-)")
    println("3. คูณ (*)")
    println("4. หาร (/)")
    
    // ใน Part นี้จะใช้ค่าคงที่ก่อน
    // (จะเรียนรับ Input จาก User ใน Part ถัดไป)
    val num1 = 10.0
    val num2 = 3.0
    val operation = "+"
    
    val result = when (operation) {
        "+" -> num1 + num2
        "-" -> num1 - num2
        "*" -> num1 * num2
        "/" -> if (num2 != 0.0) num1 / num2 else {
            println("ไม่สามารถหารด้วย 0 ได้!")
            Double.NaN
        }
        else -> {
            println("ไม่รู้จักการดำเนินการนี้")
            Double.NaN
        }
    }
    
    if (!result.isNaN()) {
        println("\n$num1 $operation $num2 = $result")
    }
}
```

ผลลัพธ์:
```
=== Kotlin Calculator ===
เลือกการคำนวณ:
1. บวก (+)
2. ลบ (-)
3. คูณ (*)
4. หาร (/)

10.0 + 3.0 = 13.0
```

---

## สิ่งที่จะเรียนใน Part 02

- ตัวแปรและชนิดข้อมูลทั้งหมดใน Kotlin
- Int, Long, Double, Float, Boolean, Char, String
- Type Inference
- Operators ทุกประเภท
- String Templates ขั้นสูง
- การแปลงชนิดข้อมูล (Type Conversion)

---

*Part 01 จบแล้ว | ไปต่อ [Part 02](../part02/README.md)*
