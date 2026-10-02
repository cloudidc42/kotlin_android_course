# Part 36: Compose - Theme, Typography, Color System
## ขั้นตอนที่ 676-700

---

## ขั้นตอนที่ 676: Material 3 Color System

Material 3 ใช้ระบบสีแบบ "Color Scheme" มีสี role ต่างๆ ที่ semantic เช่น primary, secondary, tertiary, error

```kotlin
// app/src/main/java/com/example/app/ui/theme/Color.kt
package com.example.app.ui.theme

import androidx.compose.ui.graphics.Color

// Light Theme Colors
val Purple80 = Color(0xFFD0BCFF)
val PurpleGrey80 = Color(0xFFCCC2DC)
val Pink80 = Color(0xFFEFB8C8)

// Dark Theme Colors
val Purple40 = Color(0xFF6650A4)
val PurpleGrey40 = Color(0xFF625B71)
val Pink40 = Color(0xFF7D5260)

// Custom Brand Colors
val BrandPrimary = Color(0xFF1976D2)       // น้ำเงิน
val BrandSecondary = Color(0xFF388E3C)     // เขียว
val BrandTertiary = Color(0xFFE64A19)      // ส้ม
val BrandError = Color(0xFFD32F2F)         // แดง

// Neutral Colors
val Gray50 = Color(0xFFFAFAFA)
val Gray100 = Color(0xFFF5F5F5)
val Gray200 = Color(0xFFEEEEEE)
val Gray900 = Color(0xFF212121)

// Background Colors
val LightBackground = Color(0xFFFFFBFE)
val DarkBackground = Color(0xFF1C1B1F)
val LightSurface = Color(0xFFFFFBFE)
val DarkSurface = Color(0xFF1C1B1F)
```

```kotlin
// app/src/main/java/com/example/app/ui/theme/Theme.kt
package com.example.app.ui.theme

import android.app.Activity
import android.os.Build
import androidx.compose.foundation.isSystemInDarkTheme
import androidx.compose.material3.*
import androidx.compose.runtime.Composable
import androidx.compose.runtime.SideEffect
import androidx.compose.ui.graphics.toArgb
import androidx.compose.ui.platform.LocalContext
import androidx.compose.ui.platform.LocalView
import androidx.core.view.WindowCompat

private val LightColorScheme = lightColorScheme(
    primary = BrandPrimary,
    onPrimary = Color.White,
    primaryContainer = Color(0xFFD3E4FF),
    onPrimaryContainer = Color(0xFF001C38),
    secondary = BrandSecondary,
    onSecondary = Color.White,
    tertiary = BrandTertiary,
    error = BrandError,
    background = LightBackground,
    surface = LightSurface,
)

private val DarkColorScheme = darkColorScheme(
    primary = Color(0xFF9ECAFF),
    onPrimary = Color(0xFF003258),
    primaryContainer = Color(0xFF00497E),
    onPrimaryContainer = Color(0xFFD3E4FF),
    secondary = Color(0xFF89D988),
    background = DarkBackground,
    surface = DarkSurface,
)

@Composable
fun AppTheme(
    darkTheme: Boolean = isSystemInDarkTheme(),
    dynamicColor: Boolean = true, // Android 12+
    content: @Composable () -> Unit
) {
    val colorScheme = when {
        dynamicColor && Build.VERSION.SDK_INT >= Build.VERSION_CODES.S -> {
            val context = LocalContext.current
            if (darkTheme) dynamicDarkColorScheme(context)
            else dynamicLightColorScheme(context)
        }
        darkTheme -> DarkColorScheme
        else -> LightColorScheme
    }

    val view = LocalView.current
    if (!view.isInEditMode) {
        SideEffect {
            val window = (view.context as Activity).window
            window.statusBarColor = colorScheme.primary.toArgb()
            WindowCompat.getInsetsController(window, view).isAppearanceLightStatusBars = !darkTheme
        }
    }

    MaterialTheme(
        colorScheme = colorScheme,
        typography = AppTypography,
        content = content
    )
}
```

---

## ขั้นตอนที่ 677: Typography System

```kotlin
// app/src/main/java/com/example/app/ui/theme/Type.kt
package com.example.app.ui.theme

import androidx.compose.material3.Typography
import androidx.compose.ui.text.TextStyle
import androidx.compose.ui.text.font.Font
import androidx.compose.ui.text.font.FontFamily
import androidx.compose.ui.text.font.FontWeight
import androidx.compose.ui.unit.sp

// Custom Font Family
val SarabunFontFamily = FontFamily(
    Font(R.font.sarabun_regular, FontWeight.Normal),
    Font(R.font.sarabun_medium, FontWeight.Medium),
    Font(R.font.sarabun_semibold, FontWeight.SemiBold),
    Font(R.font.sarabun_bold, FontWeight.Bold)
)

// Material 3 Typography Scale
val AppTypography = Typography(
    // Display - ขนาดใหญ่สุด สำหรับ hero text
    displayLarge = TextStyle(
        fontFamily = SarabunFontFamily,
        fontWeight = FontWeight.Normal,
        fontSize = 57.sp,
        lineHeight = 64.sp
    ),
    displayMedium = TextStyle(
        fontFamily = SarabunFontFamily,
        fontWeight = FontWeight.Normal,
        fontSize = 45.sp,
        lineHeight = 52.sp
    ),
    displaySmall = TextStyle(
        fontFamily = SarabunFontFamily,
        fontWeight = FontWeight.Normal,
        fontSize = 36.sp,
        lineHeight = 44.sp
    ),

    // Headline - หัวข้อ
    headlineLarge = TextStyle(
        fontFamily = SarabunFontFamily,
        fontWeight = FontWeight.SemiBold,
        fontSize = 32.sp,
        lineHeight = 40.sp
    ),
    headlineMedium = TextStyle(
        fontFamily = SarabunFontFamily,
        fontWeight = FontWeight.SemiBold,
        fontSize = 28.sp,
        lineHeight = 36.sp
    ),
    headlineSmall = TextStyle(
        fontFamily = SarabunFontFamily,
        fontWeight = FontWeight.SemiBold,
        fontSize = 24.sp,
        lineHeight = 32.sp
    ),

    // Title - หัวข้อย่อย
    titleLarge = TextStyle(
        fontFamily = SarabunFontFamily,
        fontWeight = FontWeight.Medium,
        fontSize = 22.sp,
        lineHeight = 28.sp
    ),
    titleMedium = TextStyle(
        fontFamily = SarabunFontFamily,
        fontWeight = FontWeight.Medium,
        fontSize = 16.sp,
        lineHeight = 24.sp
    ),
    titleSmall = TextStyle(
        fontFamily = SarabunFontFamily,
        fontWeight = FontWeight.Medium,
        fontSize = 14.sp,
        lineHeight = 20.sp
    ),

    // Body - เนื้อหา
    bodyLarge = TextStyle(
        fontFamily = SarabunFontFamily,
        fontWeight = FontWeight.Normal,
        fontSize = 16.sp,
        lineHeight = 24.sp
    ),
    bodyMedium = TextStyle(
        fontFamily = SarabunFontFamily,
        fontWeight = FontWeight.Normal,
        fontSize = 14.sp,
        lineHeight = 20.sp
    ),
    bodySmall = TextStyle(
        fontFamily = SarabunFontFamily,
        fontWeight = FontWeight.Normal,
        fontSize = 12.sp,
        lineHeight = 16.sp
    ),

    // Label - ป้าย, ปุ่ม
    labelLarge = TextStyle(
        fontFamily = SarabunFontFamily,
        fontWeight = FontWeight.Medium,
        fontSize = 14.sp,
        lineHeight = 20.sp
    ),
    labelMedium = TextStyle(
        fontFamily = SarabunFontFamily,
        fontWeight = FontWeight.Medium,
        fontSize = 12.sp,
        lineHeight = 16.sp
    ),
    labelSmall = TextStyle(
        fontFamily = SarabunFontFamily,
        fontWeight = FontWeight.Medium,
        fontSize = 11.sp,
        lineHeight = 16.sp
    )
)
```

---

## ขั้นตอนที่ 678: ใช้ Theme ใน Component

```kotlin
import androidx.compose.material3.*
import androidx.compose.runtime.Composable
import androidx.compose.ui.Modifier

@Composable
fun ThemedComponents() {
    // ใช้ MaterialTheme.colorScheme เพื่อดึงสีจาก theme
    Column(modifier = Modifier.padding(16.dp)) {
        // สีจาก theme
        Text(
            text = "Primary Text",
            color = MaterialTheme.colorScheme.primary,
            style = MaterialTheme.typography.headlineMedium
        )

        Text(
            text = "Secondary Text",
            color = MaterialTheme.colorScheme.secondary,
            style = MaterialTheme.typography.titleMedium
        )

        // Surface ใช้ theme สีโดยอัตโนมัติ
        Surface(
            color = MaterialTheme.colorScheme.primaryContainer,
            contentColor = MaterialTheme.colorScheme.onPrimaryContainer
        ) {
            Text(
                "ข้อความบน Primary Container",
                modifier = Modifier.padding(16.dp)
            )
        }

        // Card ใช้สีจาก theme
        Card(
            colors = CardDefaults.cardColors(
                containerColor = MaterialTheme.colorScheme.surfaceVariant
            )
        ) {
            Text(
                "Card สี surfaceVariant",
                modifier = Modifier.padding(16.dp),
                color = MaterialTheme.colorScheme.onSurfaceVariant
            )
        }
    }
}
```

---

## ขั้นตอนที่ 679: Dark Mode Toggle

```kotlin
import androidx.compose.foundation.layout.*
import androidx.compose.material3.*
import androidx.compose.runtime.*
import androidx.compose.ui.Modifier

// CompositionLocal สำหรับ dark mode
@Composable
fun DarkModeToggleApp() {
    var isDarkMode by remember { mutableStateOf(false) }

    AppTheme(darkTheme = isDarkMode) {
        Scaffold { padding ->
            Column(
                modifier = Modifier
                    .padding(padding)
                    .padding(16.dp)
            ) {
                Row(
                    horizontalArrangement = Arrangement.SpaceBetween,
                    modifier = Modifier.fillMaxWidth()
                ) {
                    Text("Dark Mode")
                    Switch(
                        checked = isDarkMode,
                        onCheckedChange = { isDarkMode = it }
                    )
                }

                // แสดง color palette ปัจจุบัน
                Text("Color Palette", style = MaterialTheme.typography.titleMedium)
                Spacer(Modifier.height(8.dp))

                val colors = listOf(
                    "primary" to MaterialTheme.colorScheme.primary,
                    "secondary" to MaterialTheme.colorScheme.secondary,
                    "tertiary" to MaterialTheme.colorScheme.tertiary,
                    "background" to MaterialTheme.colorScheme.background,
                    "surface" to MaterialTheme.colorScheme.surface,
                    "error" to MaterialTheme.colorScheme.error
                )

                colors.forEach { (name, color) ->
                    Row(
                        modifier = Modifier
                            .fillMaxWidth()
                            .height(40.dp)
                            .background(color),
                        verticalAlignment = Alignment.CenterVertically
                    ) {
                        Text(
                            name,
                            modifier = Modifier.padding(start = 8.dp),
                            color = Color.White
                        )
                    }
                }
            }
        }
    }
}
```

---

## ขั้นตอนที่ 680: Custom Shape System

```kotlin
// app/src/main/java/com/example/app/ui/theme/Shape.kt
package com.example.app.ui.theme

import androidx.compose.foundation.shape.RoundedCornerShape
import androidx.compose.material3.Shapes
import androidx.compose.ui.unit.dp

// Custom Shape สำหรับ app
val AppShapes = Shapes(
    // ExtraSmall - chip, badge
    extraSmall = RoundedCornerShape(4.dp),
    // Small - button, text field
    small = RoundedCornerShape(8.dp),
    // Medium - card, dialog
    medium = RoundedCornerShape(12.dp),
    // Large - bottom sheet, navigation drawer
    large = RoundedCornerShape(16.dp),
    // ExtraLarge - full screen modal
    extraLarge = RoundedCornerShape(28.dp)
)

// ใน Theme
@Composable
fun AppTheme(content: @Composable () -> Unit) {
    MaterialTheme(
        colorScheme = /* ... */,
        typography = AppTypography,
        shapes = AppShapes, // ใส่ shapes เข้าไป
        content = content
    )
}

// ใช้งาน Shape จาก Theme
@Composable
fun ShapeExamples() {
    Column(verticalArrangement = Arrangement.spacedBy(8.dp)) {
        Surface(
            shape = MaterialTheme.shapes.small,
            color = MaterialTheme.colorScheme.primaryContainer
        ) {
            Text("Shape Small (8dp)", modifier = Modifier.padding(12.dp))
        }

        Surface(
            shape = MaterialTheme.shapes.medium,
            color = MaterialTheme.colorScheme.secondaryContainer
        ) {
            Text("Shape Medium (12dp)", modifier = Modifier.padding(12.dp))
        }

        Surface(
            shape = MaterialTheme.shapes.large,
            color = MaterialTheme.colorScheme.tertiaryContainer
        ) {
            Text("Shape Large (16dp)", modifier = Modifier.padding(12.dp))
        }
    }
}
```

---

*Part 36 จบแล้ว | ก่อนหน้า: [Part 35](../part35/README.md) | ถัดไป: [Part 37](../part37/README.md)*
