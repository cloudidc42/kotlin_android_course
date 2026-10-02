# Part 96: Code Quality & Static Analysis
## ขั้นตอนที่ 1876-1900

---

## ขั้นตอนที่ 1876: Detekt Setup

```kotlin
// ============================================
// Detekt - Static Analysis for Kotlin
// ============================================

// build.gradle.kts (root)
plugins {
    id("io.gitlab.arturbosch.detekt") version "1.23.x" apply false
}

// build.gradle.kts (module)
plugins {
    id("io.gitlab.arturbosch.detekt")
}

detekt {
    config.setFrom("$rootDir/detekt-config.yml")
    baseline = file("$rootDir/detekt-baseline.xml")
    parallel = true
    buildUponDefaultConfig = true
    
    // Auto correct formatting issues
    autoCorrect = true
}

dependencies {
    detektPlugins("io.gitlab.arturbosch.detekt:detekt-formatting:1.23.x")
    detektPlugins("com.twitter.compose.rules:detekt:0.0.x")  // Compose rules
}
```

```yaml
# detekt-config.yml

complexity:
  LongMethod:
    threshold: 60
  LongParameterList:
    threshold: 7
  TooManyFunctions:
    threshold: 15
  ComplexMethod:
    threshold: 20

style:
  MagicNumber:
    active: true
    ignoreNumbers: ['-1', '0', '1', '2']
    ignorePropertyDeclaration: true
    ignoreEnums: true
    ignoreAnnotation: true
  MaxLineLength:
    maxLineLength: 120
  WildcardImport:
    active: true
    excludeImports: []
  UnusedImports:
    active: true
  ReturnCount:
    max: 4

naming:
  FunctionNaming:
    active: true
    functionPattern: '[a-z][a-zA-Z0-9]*'
    ignoreAnnotated: ['Composable']  # Composables start with uppercase
  ClassNaming:
    classPattern: '[A-Z][a-zA-Z0-9]*'

performance:
  ForEachOnRange:
    active: true
  SpreadOperator:
    active: true

exceptions:
  SwallowedException:
    active: true
  TooGenericExceptionCaught:
    active: true
  TooGenericExceptionThrown:
    active: true

comments:
  UndocumentedPublicClass:
    active: false  # KDoc optional for internal apps
  UndocumentedPublicFunction:
    active: false

compose:
  ComposableFunctionName:
    active: true
  ComposableNaming:
    active: true
  MutableStateAutoboxing:
    active: true
  ViewModelForwarding:
    active: true
  ViewModelInjection:
    active: true
```

---

## ขั้นตอนที่ 1877: ktlint Configuration

```kotlin
// ============================================
// ktlint - Kotlin Formatter
// ============================================

// build.gradle.kts (root)
plugins {
    id("org.jlleitschuh.gradle.ktlint") version "12.x" apply false
}

// build.gradle.kts (module)
plugins {
    id("org.jlleitschuh.gradle.ktlint")
}

ktlint {
    version.set("1.0.x")
    android.set(true)
    outputColorName.set("RED")
    
    filter {
        exclude("**/generated/**")
        include("**/*.kt")
    }
}

// .editorconfig (project root)
/*
[*.kt]
max_line_length = 120
indent_size = 4
insert_final_newline = true
trim_trailing_whitespace = true

ktlint_standard_import-ordering = enabled
ktlint_standard_no-wildcard-imports = enabled
ktlint_standard_function-naming = disabled  # Allow Composable naming
ktlint_compose_function-naming = enabled

[*.kts]
indent_size = 4
*/

// Run ktlint
// ./gradlew ktlintCheck  - check
// ./gradlew ktlintFormat - auto format
```

---

## ขั้นตอนที่ 1878: Git Hooks สำหรับ Quality Gates

```bash
#!/bin/bash
# .git/hooks/pre-commit

# Run ktlint before every commit
echo "Running ktlint check..."
./gradlew ktlintCheck

if [ $? -ne 0 ]; then
    echo "❌ ktlint check failed. Run ./gradlew ktlintFormat to fix."
    exit 1
fi

# Run detekt
echo "Running detekt..."
./gradlew detekt

if [ $? -ne 0 ]; then
    echo "❌ Detekt check failed. Fix issues before committing."
    exit 1
fi

echo "✅ All checks passed!"
```

```kotlin
// ============================================
// Automate with Gradle task
// ============================================

// build.gradle.kts (root)
tasks.register("codeQuality") {
    group = "verification"
    description = "Run all code quality checks"
    
    dependsOn(
        ":app:ktlintCheck",
        ":app:detekt",
        ":app:lint",
        ":app:test"
    )
}

// Setup pre-commit hook automatically
tasks.register("installGitHooks") {
    group = "setup"
    doLast {
        val hooksDir = file("${rootProject.rootDir}/.git/hooks")
        val preCommitHook = file("${hooksDir}/pre-commit")
        
        preCommitHook.writeText("""
            #!/bin/bash
            ./gradlew codeQuality
        """.trimIndent())
        
        preCommitHook.setExecutable(true)
        println("Git hooks installed!")
    }
}
```

---

## ขั้นตอนที่ 1879: Unit Test Coverage

```kotlin
// ============================================
// Jacoco Test Coverage
// ============================================

// build.gradle.kts (app)
android {
    buildTypes {
        debug {
            enableAndroidTestCoverage = true
            enableUnitTestCoverage = true
        }
    }
}

// coverage.gradle.kts
plugins {
    jacoco
}

jacoco {
    toolVersion = "0.8.10"
}

tasks.register<JacocoReport>("jacocoTestReport") {
    group = "Reporting"
    description = "Generate Jacoco coverage reports"
    
    dependsOn("testDebugUnitTest")
    
    reports {
        xml.required.set(true)
        html.required.set(true)
        csv.required.set(false)
    }
    
    val fileFilter = listOf(
        "**/R.class", "**/R$*.class",
        "**/BuildConfig.*", "**/Manifest*.*",
        "**/*Test*.*", "android/**/*.*",
        "**/*_MembersInjector*.*",
        "**/*_Factory*.*",
        "**/*Component*.*",
        "**/*Module*.*",
        "**/databinding/**",
        "**/generated/**"
    )
    
    val debugTree = fileTree("${buildDir}/tmp/kotlin-classes/debug") {
        exclude(fileFilter)
    }
    
    sourceDirectories.setFrom("src/main/java", "src/main/kotlin")
    classDirectories.setFrom(debugTree)
    executionData.setFrom(fileTree(buildDir) {
        include("jacoco/testDebugUnitTest.exec")
    })
}

// Coverage threshold
tasks.register("checkCoverage") {
    dependsOn("jacocoTestReport")
    
    doLast {
        val report = file("${buildDir}/reports/jacoco/jacocoTestReport/jacocoTestReport.xml")
        // Parse XML and verify coverage >= 80%
        // Fail build if below threshold
    }
}
```

---

## ขั้นตอนที่ 1880: SonarQube Integration

```yaml
# .github/workflows/sonarqube.yml
name: SonarQube Analysis

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  sonar:
    runs-on: ubuntu-latest
    
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0  # Full history for blame
      
      - name: Set up JDK 17
        uses: actions/setup-java@v4
        with:
          java-version: '17'
          distribution: 'temurin'
          cache: gradle
      
      - name: Run tests with coverage
        run: ./gradlew testDebugUnitTest jacocoTestReport
      
      - name: SonarQube Scan
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}
        run: |
          ./gradlew sonar \
            -Dsonar.projectKey=my-android-app \
            -Dsonar.host.url=${{ secrets.SONAR_HOST_URL }} \
            -Dsonar.login=${{ secrets.SONAR_TOKEN }}
```

```kotlin
// build.gradle.kts (root) - SonarQube plugin
plugins {
    id("org.sonarqube") version "4.x"
}

sonar {
    properties {
        property("sonar.projectKey", "myapp")
        property("sonar.projectName", "My Android App")
        property("sonar.sources", "src/main")
        property("sonar.tests", "src/test,src/androidTest")
        property("sonar.coverage.jacoco.xmlReportPaths", 
                 "${buildDir}/reports/jacoco/jacocoTestReport/jacocoTestReport.xml")
        property("sonar.kotlin.detekt.reportPaths",
                 "${buildDir}/reports/detekt/detekt.xml")
        property("sonar.androidLint.reportPaths",
                 "${buildDir}/reports/lint-results-debug.xml")
        property("sonar.exclusions",
                 "**/*Test*.kt,**/generated/**,**/build/**")
    }
}
```

---

*Part 96 จบแล้ว | ก่อนหน้า: [Part 95](../part95/README.md) | ถัดไป: [Part 97](../part97/README.md)*
