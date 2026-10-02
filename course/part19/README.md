# Part 19: Kotlin DSL และ Builder Pattern
## ขั้นตอนที่ 451-475

---

## ขั้นตอนที่ 451: DSL คืออะไร?

DSL (Domain-Specific Language) คือ mini-language ที่ออกแบบสำหรับ domain เฉพาะ

```kotlin
// ตัวอย่าง DSL ที่เราเคยใช้:

// Gradle Kotlin DSL
plugins {
    alias(libs.plugins.android.application)
    alias(libs.plugins.kotlin.android)
}

android {
    compileSdk = 35
    defaultConfig {
        applicationId = "com.example.app"
        minSdk = 26
    }
}

// HTML DSL (kotlinx.html)
html {
    head { title("My Page") }
    body {
        h1 { +"Hello World" }
        p { +"This is DSL" }
    }
}

// Jetpack Compose ก็เป็น DSL!
@Composable
fun MyUI() {
    Column {
        Text("Hello")
        Row {
            Button(onClick = {}) { Text("Click") }
        }
    }
}
```

---

## ขั้นตอนที่ 452: Builder Pattern

```kotlin
// ============================================
// Traditional Builder Pattern
// ============================================

class AlertDialog private constructor(
    val title: String,
    val message: String,
    val positiveButton: String?,
    val negativeButton: String?,
    val cancelable: Boolean,
    val onPositive: (() -> Unit)?,
    val onNegative: (() -> Unit)?
) {
    class Builder {
        private var title: String = ""
        private var message: String = ""
        private var positiveButton: String? = null
        private var negativeButton: String? = null
        private var cancelable: Boolean = true
        private var onPositive: (() -> Unit)? = null
        private var onNegative: (() -> Unit)? = null
        
        fun title(title: String) = apply { this.title = title }
        fun message(message: String) = apply { this.message = message }
        fun positiveButton(text: String, onClick: () -> Unit) = apply {
            positiveButton = text
            onPositive = onClick
        }
        fun negativeButton(text: String, onClick: () -> Unit) = apply {
            negativeButton = text
            onNegative = onClick
        }
        fun cancelable(value: Boolean) = apply { cancelable = value }
        
        fun build() = AlertDialog(
            title, message, positiveButton, negativeButton,
            cancelable, onPositive, onNegative
        )
    }
}

// ใช้งาน
val dialog = AlertDialog.Builder()
    .title("ยืนยันการลบ")
    .message("คุณต้องการลบข้อมูลนี้ใช่หรือไม่?")
    .positiveButton("ลบ") { println("Deleted") }
    .negativeButton("ยกเลิก") { println("Cancelled") }
    .cancelable(false)
    .build()
```

---

## ขั้นตอนที่ 453: Kotlin DSL Builder

```kotlin
// ============================================
// Kotlin DSL ด้วย Lambda with Receiver
// ============================================

// DSL ที่สะอาดกว่า Builder
class AlertDialogDsl {
    var title: String = ""
    var message: String = ""
    var positiveButton: String? = null
    var negativeButton: String? = null
    var cancelable: Boolean = true
    var onPositive: (() -> Unit)? = null
    var onNegative: (() -> Unit)? = null
    
    fun positiveButton(text: String, onClick: () -> Unit) {
        positiveButton = text
        onPositive = onClick
    }
    
    fun negativeButton(text: String, onClick: () -> Unit) {
        negativeButton = text
        onNegative = onClick
    }
    
    internal fun build() = AlertDialogConfig(
        title = title,
        message = message,
        positiveButton = positiveButton,
        negativeButton = negativeButton,
        cancelable = cancelable,
        onPositive = onPositive,
        onNegative = onNegative
    )
}

data class AlertDialogConfig(
    val title: String,
    val message: String,
    val positiveButton: String?,
    val negativeButton: String?,
    val cancelable: Boolean,
    val onPositive: (() -> Unit)?,
    val onNegative: (() -> Unit)?
)

// DSL function
fun alertDialog(block: AlertDialogDsl.() -> Unit): AlertDialogConfig {
    return AlertDialogDsl().apply(block).build()
}

// ใช้งาน - สะอาดกว่ามาก!
val config = alertDialog {
    title = "ยืนยันการลบ"
    message = "คุณต้องการลบข้อมูลนี้ใช่หรือไม่?"
    positiveButton("ลบ") { println("Deleted") }
    negativeButton("ยกเลิก") { println("Cancelled") }
    cancelable = false
}
```

---

## ขั้นตอนที่ 454: HTTP Request DSL

```kotlin
// ============================================
// HTTP Request DSL
// ============================================

@DslMarker
annotation class RequestDsl

@RequestDsl
class RequestBuilder {
    var url: String = ""
    var method: String = "GET"
    val headers = mutableMapOf<String, String>()
    var body: String? = null
    var timeout: Int = 30000
    
    fun header(key: String, value: String) {
        headers[key] = value
    }
    
    fun bearer(token: String) {
        header("Authorization", "Bearer $token")
    }
    
    fun contentType(type: String) {
        header("Content-Type", type)
    }
    
    fun json(block: JsonBuilder.() -> Unit) {
        contentType("application/json")
        body = JsonBuilder().apply(block).build()
    }
    
    fun build() = Request(url, method, headers, body, timeout)
}

@RequestDsl
class JsonBuilder {
    private val fields = mutableMapOf<String, Any?>()
    
    infix fun String.to(value: Any?) { fields[this] = value }
    
    fun build(): String = Gson().toJson(fields)
}

data class Request(
    val url: String,
    val method: String,
    val headers: Map<String, String>,
    val body: String?,
    val timeout: Int
)

// DSL functions
fun get(url: String, block: RequestBuilder.() -> Unit = {}): Request {
    return RequestBuilder().apply { this.url = url; method = "GET"; block() }.build()
}

fun post(url: String, block: RequestBuilder.() -> Unit = {}): Request {
    return RequestBuilder().apply { this.url = url; method = "POST"; block() }.build()
}

fun put(url: String, block: RequestBuilder.() -> Unit = {}): Request {
    return RequestBuilder().apply { this.url = url; method = "PUT"; block() }.build()
}

// ใช้งาน
val getRequest = get("https://api.example.com/users") {
    bearer("my-token-123")
    timeout = 5000
}

val postRequest = post("https://api.example.com/users") {
    bearer("my-token-123")
    json {
        "name" to "Alice"
        "email" to "alice@example.com"
        "age" to 25
        "active" to true
    }
}

println(postRequest.body)
// {"name":"Alice","email":"alice@example.com","age":25,"active":true}
```

---

## ขั้นตอนที่ 455: UI DSL

```kotlin
// ============================================
// Simple HTML DSL
// ============================================

interface HtmlElement {
    fun render(): String
}

class TextElement(private val text: String) : HtmlElement {
    override fun render() = text
}

@HtmlDsl
class TagElement(private val tag: String) : HtmlElement {
    val attributes = mutableMapOf<String, String>()
    val children = mutableListOf<HtmlElement>()
    
    operator fun String.unaryPlus() {
        children.add(TextElement(this))
    }
    
    fun attribute(key: String, value: String) {
        attributes[key] = value
    }
    
    override fun render(): String {
        val attrs = if (attributes.isEmpty()) ""
        else " " + attributes.entries.joinToString(" ") { "${it.key}=\"${it.value}\"" }
        
        val content = children.joinToString("") { it.render() }
        return "<$tag$attrs>$content</$tag>"
    }
}

@DslMarker
annotation class HtmlDsl

@HtmlDsl
class HtmlBuilder : TagElement("html")

fun html(block: HtmlBuilder.() -> Unit): String {
    return "<!DOCTYPE html>\n" + HtmlBuilder().apply(block).render()
}

fun TagElement.head(block: TagElement.() -> Unit): TagElement {
    return TagElement("head").apply(block).also { children.add(it) }
}

fun TagElement.body(block: TagElement.() -> Unit): TagElement {
    return TagElement("body").apply(block).also { children.add(it) }
}

fun TagElement.h1(block: TagElement.() -> Unit): TagElement {
    return TagElement("h1").apply(block).also { children.add(it) }
}

fun TagElement.p(block: TagElement.() -> Unit): TagElement {
    return TagElement("p").apply(block).also { children.add(it) }
}

fun TagElement.a(href: String, block: TagElement.() -> Unit): TagElement {
    return TagElement("a").apply {
        attribute("href", href)
        block()
    }.also { children.add(it) }
}

fun TagElement.title(block: TagElement.() -> Unit): TagElement {
    return TagElement("title").apply(block).also { children.add(it) }
}

// ใช้งาน
val page = html {
    head {
        title { +"Kotlin DSL Demo" }
    }
    body {
        h1 { +"สวัสดี โลก!" }
        p { +"นี่คือตัวอย่าง HTML DSL ใน Kotlin" }
        p {
            +"ดูตัวอย่างเพิ่มเติมที่ "
            a("https://kotlinlang.org") { +"Kotlin" }
        }
    }
}

println(page)
```

---

## ขั้นตอนที่ 456: Configuration DSL

```kotlin
// ============================================
// App Configuration DSL
// ============================================

@DslMarker
annotation class ConfigDsl

@ConfigDsl
class AppConfig {
    var appName: String = ""
    var version: String = "1.0.0"
    
    val database = DatabaseConfig()
    val network = NetworkConfig()
    val features = FeatureFlags()
    val logging = LoggingConfig()
    
    fun database(block: DatabaseConfig.() -> Unit) {
        database.apply(block)
    }
    
    fun network(block: NetworkConfig.() -> Unit) {
        network.apply(block)
    }
    
    fun features(block: FeatureFlags.() -> Unit) {
        features.apply(block)
    }
    
    fun logging(block: LoggingConfig.() -> Unit) {
        logging.apply(block)
    }
}

@ConfigDsl
class DatabaseConfig {
    var host: String = "localhost"
    var port: Int = 5432
    var name: String = "app_db"
    var poolSize: Int = 10
    var timeout: Int = 30000
}

@ConfigDsl
class NetworkConfig {
    var baseUrl: String = ""
    var timeout: Int = 30000
    var retries: Int = 3
    var enableLogging: Boolean = false
}

@ConfigDsl
class FeatureFlags {
    private val flags = mutableMapOf<String, Boolean>()
    
    fun enable(feature: String) { flags[feature] = true }
    fun disable(feature: String) { flags[feature] = false }
    fun isEnabled(feature: String) = flags[feature] ?: false
}

@ConfigDsl
class LoggingConfig {
    var level: String = "INFO"
    var format: String = "text"
    var file: String? = null
}

fun appConfig(block: AppConfig.() -> Unit): AppConfig {
    return AppConfig().apply(block)
}

// ใช้งาน
val config = appConfig {
    appName = "My Kotlin App"
    version = "2.0.0"
    
    database {
        host = "db.example.com"
        port = 5432
        name = "production_db"
        poolSize = 20
    }
    
    network {
        baseUrl = "https://api.example.com"
        timeout = 10000
        retries = 3
        enableLogging = true
    }
    
    features {
        enable("dark_mode")
        enable("analytics")
        disable("beta_features")
    }
    
    logging {
        level = "DEBUG"
        format = "json"
        file = "/var/log/app.log"
    }
}

println("App: ${config.appName} v${config.version}")
println("DB: ${config.database.host}:${config.database.port}/${config.database.name}")
println("Dark Mode: ${config.features.isEnabled("dark_mode")}")
```

---

## แบบฝึกหัด Part 19

```kotlin
// แบบฝึกหัด: Email DSL
// สร้าง DSL สำหรับสร้าง Email

@DslMarker
annotation class EmailDsl

data class EmailMessage(
    val from: String,
    val to: List<String>,
    val cc: List<String>,
    val bcc: List<String>,
    val subject: String,
    val body: String,
    val isHtml: Boolean,
    val attachments: List<String>
)

@EmailDsl
class EmailBuilder {
    var from: String = ""
    var subject: String = ""
    var body: String = ""
    var isHtml: Boolean = false
    private val to = mutableListOf<String>()
    private val cc = mutableListOf<String>()
    private val bcc = mutableListOf<String>()
    private val attachments = mutableListOf<String>()
    
    fun to(vararg emails: String) { to.addAll(emails) }
    fun cc(vararg emails: String) { cc.addAll(emails) }
    fun bcc(vararg emails: String) { bcc.addAll(emails) }
    fun attach(filePath: String) { attachments.add(filePath) }
    
    fun htmlBody(block: TagElement.() -> Unit) {
        isHtml = true
        body = TagElement("div").apply(block).render()
    }
    
    fun build() = EmailMessage(from, to, cc, bcc, subject, body, isHtml, attachments)
}

fun email(block: EmailBuilder.() -> Unit): EmailMessage {
    return EmailBuilder().apply(block).build()
}

// ตัวอย่างการใช้งานที่ต้องการ:
val message = email {
    from = "sender@example.com"
    to("alice@example.com", "bob@example.com")
    cc("manager@example.com")
    subject = "รายงานประจำวัน"
    htmlBody {
        h1 { +"รายงาน" }
        p { +"เนื้อหารายงาน..." }
    }
    attach("/reports/daily.pdf")
}

// TODO: ใช้ EmailBuilder ด้านบนให้ครบและ test
```

---

*Part 19 จบแล้ว | ก่อนหน้า: [Part 18](../part18/README.md) | ถัดไป: [Part 20](../part20/README.md)*
