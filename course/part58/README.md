# Part 58: Accessibility & Localization
## ขั้นตอนที่ 926-950

---

## ขั้นตอนที่ 926: Accessibility คืออะไร?

```
Accessibility (a11y) - การทำให้ app ใช้งานได้สำหรับทุกคน

ผู้ใช้ที่ต้องการ Accessibility:
- ผู้พิการทางสายตา: ใช้ TalkBack screen reader
- ผู้พิการทางการได้ยิน: ต้องการ captions, visual cues
- ผู้พิการทางการเคลื่อนไหว: ใช้ switch access, external keyboard
- ผู้สูงอายุ: ต้องการ font ใหญ่, contrast สูง

Android Accessibility Services:
- TalkBack: screen reader
- Switch Access: ใช้ switch devices
- Voice Access: สั่งงานด้วยเสียง
- Magnification: ขยายหน้าจอ
```

---

## ขั้นตอนที่ 927: Compose Accessibility

```kotlin
// ============================================
// contentDescription - สำคัญมาก!
// ============================================

// ❌ ไม่มี content description
Image(
    painter = painterResource(R.drawable.product_image),
    contentDescription = null  // ❌ Screen reader จะพูดว่า "unlabeled"
)

// ✅ มี content description
Image(
    painter = painterResource(R.drawable.product_image),
    contentDescription = "รูปสินค้า: ${product.name}"
)

// Decorative images - null is OK
Image(
    painter = painterResource(R.drawable.background_decoration),
    contentDescription = null  // ✅ null สำหรับ decorative elements
)

// ============================================
// semantics - ควบคุม accessibility tree
// ============================================

@Composable
fun ProductCard(product: Product, onClick: () -> Unit) {
    Card(
        modifier = Modifier
            .semantics(mergeDescendants = true) {
                // Merge children semantics เป็น node เดียว
                onClick(label = "ดูรายละเอียด ${product.name}") {
                    onClick()
                    true
                }
            }
            .clickable(onClick = onClick)
    ) {
        Column {
            AsyncImage(
                model = product.imageUrl,
                contentDescription = "รูป${product.name}"
            )
            Text(product.name)
            Text("฿${product.price}")
            
            if (product.isOnSale) {
                Text(
                    "ลดราคา",
                    modifier = Modifier.semantics {
                        stateDescription = "สินค้าลดราคา ${product.discountPercent}%"
                    }
                )
            }
        }
    }
}

// ============================================
// Custom Actions
// ============================================

@Composable
fun SwipeableListItem(
    item: Todo,
    onComplete: () -> Unit,
    onDelete: () -> Unit
) {
    Box(
        modifier = Modifier.semantics {
            customActions = listOf(
                CustomAccessibilityAction("ทำเครื่องหมายเสร็จสิ้น") {
                    onComplete()
                    true
                },
                CustomAccessibilityAction("ลบ") {
                    onDelete()
                    true
                }
            )
        }
    ) {
        // swipe UI
    }
}
```

---

## ขั้นตอนที่ 928: Localization

```kotlin
// res/values/strings.xml (default - English or Thai)
// <resources>
//     <string name="app_name">My App</string>
//     <string name="greeting">Hello, %1$s!</string>
//     <string name="items_count">%1$d items</string>
//     <plurals name="item_count">
//         <item quantity="one">%1$d item</item>
//         <item quantity="other">%1$d items</item>
//     </plurals>
// </resources>

// res/values-th/strings.xml (Thai)
// <resources>
//     <string name="app_name">แอปของฉัน</string>
//     <string name="greeting">สวัสดี, %1$s!</string>
//     <string name="items_count">%1$d รายการ</string>
//     <plurals name="item_count">
//         <item quantity="other">%1$d รายการ</item>
//     </plurals>
// </resources>

// ใน Compose
@Composable
fun LocalizedScreen(name: String, count: Int) {
    Column {
        // Simple string
        Text(stringResource(R.string.app_name))
        
        // With format arg
        Text(stringResource(R.string.greeting, name))
        
        // Plurals
        Text(pluralStringResource(R.plurals.item_count, count, count))
        
        // Dynamic strings
        val context = LocalContext.current
        val message = remember(name) {
            context.getString(R.string.greeting, name)
        }
        Text(message)
    }
}
```

---

## ขั้นตอนที่ 929: RTL Support

```kotlin
// ============================================
// RTL (Right-to-Left) Layout Support
// ============================================

// AndroidManifest.xml
// android:supportsRtl="true"

// Compose รองรับ RTL อัตโนมัติ
@Composable
fun RtlAwareLayout() {
    val isRtl = LocalLayoutDirection.current == LayoutDirection.Rtl
    
    Row {
        // Start = ซ้ายใน LTR, ขวาใน RTL
        Icon(
            Icons.Default.ArrowBack,
            contentDescription = null,
            modifier = Modifier.mirrored()  // กลับทิศใน RTL
        )
        
        // padding ที่รองรับ RTL
        Text(
            "Hello",
            modifier = Modifier.padding(start = 16.dp)  // start = leading edge
        )
    }
}

// Modifier ที่รองรับ RTL
val rtlModifier = Modifier
    .padding(start = 16.dp, end = 8.dp)  // ใช้ start/end แทน left/right
    .absolutePadding(left = 0.dp)  // absolute = ไม่สนใจ RTL

// ============================================
// Date, Number, Currency Formatting
// ============================================

object LocaleFormatter {
    
    fun formatDate(timestamp: Long, locale: Locale = Locale.getDefault()): String {
        return DateFormat.getDateInstance(DateFormat.MEDIUM, locale)
            .format(Date(timestamp))
    }
    
    fun formatCurrency(amount: Double, locale: Locale = Locale.getDefault()): String {
        return NumberFormat.getCurrencyInstance(locale).format(amount)
    }
    
    fun formatNumber(number: Long, locale: Locale = Locale.getDefault()): String {
        return NumberFormat.getNumberInstance(locale).format(number)
    }
    
    fun formatPercent(ratio: Double, locale: Locale = Locale.getDefault()): String {
        return NumberFormat.getPercentInstance(locale).format(ratio)
    }
}

// ใช้งาน
val dateStr = LocaleFormatter.formatDate(System.currentTimeMillis(), Locale("th", "TH"))
val priceStr = LocaleFormatter.formatCurrency(1234.56, Locale("th", "TH"))
// ราคา: ฿1,234.56
```

---

## ขั้นตอนที่ 930: Dark Theme

```kotlin
// ============================================
// Material 3 Dark Theme
// ============================================

private val DarkColorScheme = darkColorScheme(
    primary = Purple80,
    secondary = PurpleGrey80,
    tertiary = Pink80,
    background = Color(0xFF1C1B1F),
    surface = Color(0xFF1C1B1F),
    onPrimary = Color.Black,
    onBackground = Color(0xFFE6E1E5),
    onSurface = Color(0xFFE6E1E5)
)

private val LightColorScheme = lightColorScheme(
    primary = Purple40,
    secondary = PurpleGrey40,
    tertiary = Pink40,
    background = Color(0xFFFFFBFE),
    surface = Color(0xFFFFFBFE),
    onPrimary = Color.White,
    onBackground = Color(0xFF1C1B1F),
    onSurface = Color(0xFF1C1B1F)
)

@Composable
fun AppTheme(
    darkTheme: Boolean = isSystemInDarkTheme(),
    dynamicColor: Boolean = true,
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
    
    // Status bar สีตาม theme
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
        typography = Typography,
        content = content
    )
}

// ViewModel สำหรับ Theme
@HiltViewModel
class ThemeViewModel @Inject constructor(
    private val userPreferences: UserPreferences
) : ViewModel() {
    
    val darkTheme: StateFlow<Boolean> = userPreferences
        .isDarkTheme
        .stateIn(viewModelScope, SharingStarted.Eagerly, false)
    
    fun toggleDarkTheme() {
        viewModelScope.launch {
            userPreferences.setDarkTheme(!darkTheme.value)
        }
    }
}
```

---

## แบบฝึกหัด Part 58

```kotlin
// แบบฝึกหัด: เพิ่ม Accessibility ให้ Shopping App

// TODO:
// 1. เพิ่ม contentDescription ให้รูปสินค้าทุกรูป
// 2. สร้าง semantic merging สำหรับ product card
// 3. เพิ่ม custom accessibility action: "เพิ่มลงตะกร้า", "เพิ่มในรายการโปรด"
// 4. เพิ่ม strings.xml ภาษาไทย
// 5. ทดสอบด้วย TalkBack (Settings > Accessibility > TalkBack)

@Composable
fun AccessibleProductCard(
    product: Product,
    onAddToCart: () -> Unit,
    onAddToWishlist: () -> Unit,
    onClick: () -> Unit
) {
    Card(
        modifier = Modifier
            .semantics(mergeDescendants = true) {
                // TODO: เพิ่ม semantics ที่เหมาะสม
                contentDescription = TODO("สร้าง description ที่ครอบคลุม")
                customActions = TODO("เพิ่ม actions")
            }
            .clickable(
                onClickLabel = "ดูรายละเอียด ${product.name}",
                onClick = onClick
            )
    ) {
        // card content
    }
}
```

---

*Part 58 จบแล้ว | ก่อนหน้า: [Part 57](../part57/README.md) | ถัดไป: [Part 59](../part59/README.md)*
