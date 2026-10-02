# Part 76: Kotlin Symbol Processing (KSP)
## ขั้นตอนที่ 1376-1400

---

## ขั้นตอนที่ 1376: KSP Overview

```
Kotlin Symbol Processing (KSP):
- Code generation tool (เหมือน kapt แต่เร็วกว่า 2x)
- อ่าน Kotlin symbols ระหว่าง compile time
- สร้าง boilerplate code อัตโนมัติ

Use Cases:
- Room generates DAO implementations
- Hilt generates DI code
- Moshi generates JSON adapters
- Custom annotation processors

KSP vs KAPT:
- KSP: เข้าใจ Kotlin directly (faster, Kotlin-first)
- KAPT: แปลง Kotlin → Java stub ก่อน (slower)
- Room, Hilt, Moshi ล้วน support KSP
```

---

## ขั้นตอนที่ 1377: Custom Annotation

```kotlin
// ============================================
// 1. Define Annotation
// ============================================

// annotations/src/main/kotlin/
@Target(AnnotationTarget.CLASS)
@Retention(AnnotationRetention.SOURCE)
annotation class AutoBuilder

@Target(AnnotationTarget.PROPERTY)
@Retention(AnnotationRetention.SOURCE)
annotation class BuilderField(
    val defaultValue: String = "",
    val required: Boolean = true
)

// เป้าหมาย: @AutoBuilder สร้าง Builder pattern อัตโนมัติ
@AutoBuilder
data class User(
    @BuilderField(required = true) val name: String,
    @BuilderField(required = true) val email: String,
    @BuilderField(defaultValue = "0") val age: Int,
    @BuilderField(defaultValue = "\"\"") val phone: String
)

// Generated code จะมีลักษณะ:
// class UserBuilder {
//     private var name: String? = null
//     private var email: String? = null
//     private var age: Int = 0
//     private var phone: String = ""
//     
//     fun name(value: String) = apply { this.name = value }
//     fun email(value: String) = apply { this.email = value }
//     fun age(value: Int) = apply { this.age = value }
//     fun phone(value: String) = apply { this.phone = value }
//     
//     fun build(): User {
//         return User(
//             name = name ?: error("name is required"),
//             email = email ?: error("email is required"),
//             age = age,
//             phone = phone
//         )
//     }
// }
```

---

## ขั้นตอนที่ 1378: KSP Processor Setup

```kotlin
// processor/build.gradle.kts
plugins {
    kotlin("jvm")
}

dependencies {
    implementation("com.google.devtools.ksp:symbol-processing-api:2.x")
    implementation(project(":annotations"))
}

// Register processor
// processor/src/main/resources/META-INF/services/
// com.google.devtools.ksp.processing.SymbolProcessorProvider
// → com.myapp.processor.AutoBuilderProcessorProvider
```

---

## ขั้นตอนที่ 1379: KSP Processor Implementation

```kotlin
// processor/src/main/kotlin/

class AutoBuilderProcessorProvider : SymbolProcessorProvider {
    override fun create(environment: SymbolProcessorEnvironment): SymbolProcessor {
        return AutoBuilderProcessor(
            codeGenerator = environment.codeGenerator,
            logger = environment.logger
        )
    }
}

class AutoBuilderProcessor(
    private val codeGenerator: CodeGenerator,
    private val logger: KSPLogger
) : SymbolProcessor {
    
    override fun process(resolver: Resolver): List<KSAnnotated> {
        val symbols = resolver
            .getSymbolsWithAnnotation(AutoBuilder::class.qualifiedName!!)
            .filterIsInstance<KSClassDeclaration>()
        
        // Classes that couldn't be processed (deferred)
        val unprocessed = mutableListOf<KSAnnotated>()
        
        symbols.forEach { classDeclaration ->
            if (!classDeclaration.validate()) {
                unprocessed.add(classDeclaration)
                return@forEach
            }
            generateBuilder(classDeclaration)
        }
        
        return unprocessed
    }
    
    private fun generateBuilder(classDeclaration: KSClassDeclaration) {
        val packageName = classDeclaration.packageName.asString()
        val className = classDeclaration.simpleName.asString()
        val builderName = "${className}Builder"
        
        val properties = classDeclaration.getAllProperties()
            .filter { it.isAnnotationPresent(BuilderField::class) }
            .toList()
        
        val fileSpec = buildString {
            appendLine("package $packageName")
            appendLine()
            appendLine("class $builderName {")
            
            // Private fields
            properties.forEach { prop ->
                val annotation = prop.getAnnotationsByType(BuilderField::class).first()
                val type = prop.type.resolve().declaration.simpleName.asString()
                val default = annotation.defaultValue.ifEmpty { "null" }
                
                if (annotation.required) {
                    appendLine("    private var ${prop.simpleName.asString()}: $type? = null")
                } else {
                    appendLine("    private var ${prop.simpleName.asString()}: $type = $default")
                }
            }
            
            appendLine()
            
            // Setter functions
            properties.forEach { prop ->
                val name = prop.simpleName.asString()
                val type = prop.type.resolve().declaration.simpleName.asString()
                appendLine("    fun $name(value: $type) = apply { this.$name = value }")
            }
            
            appendLine()
            
            // Build function
            appendLine("    fun build(): $className {")
            appendLine("        return $className(")
            
            properties.forEach { prop ->
                val name = prop.simpleName.asString()
                val annotation = prop.getAnnotationsByType(BuilderField::class).first()
                
                if (annotation.required) {
                    appendLine("            $name = $name ?: error(\"$name is required\"),")
                } else {
                    appendLine("            $name = $name,")
                }
            }
            
            appendLine("        )")
            appendLine("    }")
            appendLine("}")
        }
        
        val file = codeGenerator.createNewFile(
            dependencies = Dependencies(false, classDeclaration.containingFile!!),
            packageName = packageName,
            fileName = builderName
        )
        
        file.bufferedWriter().use { writer ->
            writer.write(fileSpec)
        }
    }
}
```

---

## ขั้นตอนที่ 1380: Using KSP Processor

```kotlin
// app/build.gradle.kts
plugins {
    id("com.google.devtools.ksp")
}

dependencies {
    implementation(project(":annotations"))
    ksp(project(":processor"))
}

// ใช้งาน
@AutoBuilder
data class Product(
    @BuilderField val name: String,
    @BuilderField val price: Double,
    @BuilderField(defaultValue = "0", required = false) val stock: Int,
    @BuilderField(defaultValue = "\"\"", required = false) val description: String
)

// Generated code ถูกสร้างอัตโนมัติ
fun createProduct(): Product {
    return ProductBuilder()
        .name("iPhone 16")
        .price(35000.0)
        .description("Apple iPhone 16")
        .build()
}

// ============================================
// Practical Example: Auto Event Tracking
// ============================================

@Target(AnnotationTarget.CLASS)
@Retention(AnnotationRetention.SOURCE)
annotation class TrackEvent(val eventName: String)

@TrackEvent("product_viewed")
data class ProductViewedEvent(
    val productId: Long,
    val productName: String,
    val price: Double,
    val category: String
)

// KSP generates:
// fun ProductViewedEvent.toAnalyticsMap(): Map<String, Any> {
//     return mapOf(
//         "event_name" to "product_viewed",
//         "product_id" to productId,
//         "product_name" to productName,
//         "price" to price,
//         "category" to category
//     )
// }
```

---

## แบบฝึกหัด Part 76

```kotlin
// แบบฝึกหัด: สร้าง @Singleton annotation processor

// 1. สร้าง @AutoModule annotation
// @AutoModule จะ generate Hilt @Module + @Provides automatically

@Target(AnnotationTarget.CLASS)
@Retention(AnnotationRetention.SOURCE)
annotation class AutoModule(
    val scope: KClass<out Annotation> = SingletonComponent::class
)

// เมื่อใส่ @AutoModule ใน class
@AutoModule
class UserRepository @Inject constructor(
    private val dao: UserDao,
    private val api: UserApi
)

// KSP ควร generate:
// @Module
// @InstallIn(SingletonComponent::class)
// abstract class UserRepositoryModule {
//     @Binds
//     @Singleton
//     abstract fun bindUserRepository(impl: UserRepository): UserRepository
// }

// 2. สร้าง KSP processor ที่ generate module นี้
class AutoModuleProcessor(
    private val codeGenerator: CodeGenerator
) : SymbolProcessor {
    override fun process(resolver: Resolver): List<KSAnnotated> {
        // TODO: implement
        return emptyList()
    }
}
```

---

*Part 76 จบแล้ว | ก่อนหน้า: [Part 75](../part75/README.md) | ถัดไป: [Part 77](../part77/README.md)*
