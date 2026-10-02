# Part 99: Accessibility & Localization
## ขั้นตอนที่ 1951-1975

---

## ขั้นตอนที่ 1951: Accessibility (a11y) ใน Compose

```kotlin
// ============================================
// Accessibility in Jetpack Compose
// ============================================

// 1. Content descriptions for images
@Composable
fun ProductImage(product: Product) {
    AsyncImage(
        model = product.imageUrl,
        contentDescription = "${product.name}, ราคา ${product.price.toCurrency()}",
        // Don't use empty string - screen reader will say "unlabeled"
        // Use null only for purely decorative images
    )
}

// Decorative image
@Composable
fun DecorativeDivider() {
    Image(
        painter = painterResource(R.drawable.decorative_wave),
        contentDescription = null  // null = screen reader ignores
    )
}

// 2. Semantic roles
@Composable
fun ToggleButton(isSelected: Boolean, onClick: () -> Unit, label: String) {
    Box(
        modifier = Modifier
            .toggleable(
                value = isSelected,
                role = Role.Checkbox,
                onValueChange = { onClick() }
            )
            .semantics {
                contentDescription = label
                stateDescription = if (isSelected) "เลือกแล้ว" else "ไม่ได้เลือก"
            }
    ) {
        // content
    }
}

// 3. Semantic grouping - merge children semantics
@Composable
fun ProductCard(product: Product, onClick: () -> Unit) {
    Card(
        onClick = onClick,
        modifier = Modifier.semantics(mergeDescendants = true) {
            // Children semantics merged into this node
            // Screen reader reads: "iPhone 16, ฿35,000, 4.8 ดาว, ปุ่มเพิ่มลงตะกร้า"
        }
    ) {
        Row {
            ProductImage(product)
            Column {
                Text(product.name)
                Text(product.price.toCurrency())
                RatingStars(product.rating)
            }
        }
    }
}

// 4. Custom accessibility actions
@Composable
fun SwipeableItem(onDelete: () -> Unit, onArchive: () -> Unit) {
    Box(
        modifier = Modifier.semantics {
            customActions = listOf(
                CustomAccessibilityAction(
                    label = "ลบรายการ",
                    action = { onDelete(); true }
                ),
                CustomAccessibilityAction(
                    label = "เก็บเข้าคลัง",
                    action = { onArchive(); true }
                )
            )
        }
    ) {
        // swipe-able content
    }
}

// 5. Focus management
@Composable
fun LoginForm() {
    val focusRequester = remember { FocusRequester() }
    val focusManager = LocalFocusManager.current
    
    // Auto-focus first field
    LaunchedEffect(Unit) {
        focusRequester.requestFocus()
    }
    
    TextField(
        value = email,
        onValueChange = { email = it },
        label = { Text("อีเมล") },
        modifier = Modifier.focusRequester(focusRequester),
        keyboardOptions = KeyboardOptions(
            keyboardType = KeyboardType.Email,
            imeAction = ImeAction.Next
        )
    )
    
    TextField(
        value = password,
        onValueChange = { password = it },
        label = { Text("รหัสผ่าน") },
        keyboardOptions = KeyboardOptions(
            keyboardType = KeyboardType.Password,
            imeAction = ImeAction.Done
        ),
        keyboardActions = KeyboardActions(
            onDone = { focusManager.clearFocus() }
        )
    )
}

// 6. Minimum touch target size (48x48 dp)
@Composable
fun SmallIconButton(icon: ImageVector, onClick: () -> Unit) {
    IconButton(
        onClick = onClick,
        modifier = Modifier
            .size(48.dp)  // ✅ minimum 48dp touch target
            .semantics {
                contentDescription = "ปิด"
            }
    ) {
        Icon(
            icon,
            contentDescription = null,  // null here since parent has description
            modifier = Modifier.size(24.dp)  // Icon can be smaller
        )
    }
}
```

---

## ขั้นตอนที่ 1952: Localization (i18n)

```kotlin
// ============================================
// String Resources & Plurals
// ============================================

// res/values/strings.xml (ภาษาไทย - default)
/*
<resources>
    <string name="app_name">แอพช้อปปิ้ง</string>
    <string name="products_title">สินค้า</string>
    <string name="search_hint">ค้นหาสินค้า</string>
    <string name="price_format">฿%1$.2f</string>
    <string name="cart_empty">ตะกร้าของคุณว่างเปล่า</string>
    
    <!-- Plurals -->
    <plurals name="item_count">
        <item quantity="one">%d รายการ</item>
        <item quantity="other">%d รายการ</item>
    </plurals>
    
    <!-- Format with args -->
    <string name="welcome_message">ยินดีต้อนรับ, %1$s!</string>
    <string name="order_status">คำสั่งซื้อ #%1$s: %2$s</string>
</resources>
*/

// res/values-en/strings.xml (English)
/*
<resources>
    <string name="app_name">Shopping App</string>
    <string name="products_title">Products</string>
    <string name="search_hint">Search products</string>
    <string name="price_format">฿%1$.2f</string>
    <string name="cart_empty">Your cart is empty</string>
    
    <plurals name="item_count">
        <item quantity="one">%d item</item>
        <item quantity="other">%d items</item>
    </plurals>
    
    <string name="welcome_message">Welcome, %1$s!</string>
    <string name="order_status">Order #%1$s: %2$s</string>
</resources>
*/

// Usage in Compose
@Composable
fun CartHeader(itemCount: Int) {
    val title = pluralStringResource(
        R.plurals.item_count,
        itemCount,
        itemCount
    )
    Text(title)
}

@Composable
fun WelcomeMessage(userName: String) {
    Text(stringResource(R.string.welcome_message, userName))
}

// ============================================
// Number & Currency Formatting
// ============================================

object LocaleFormatter {
    
    fun formatCurrency(amount: Double, locale: Locale = Locale("th", "TH")): String {
        return NumberFormat.getCurrencyInstance(locale).format(amount)
    }
    
    fun formatNumber(number: Long, locale: Locale = Locale("th", "TH")): String {
        return NumberFormat.getNumberInstance(locale).format(number)
    }
    
    fun formatPercent(value: Double, locale: Locale = Locale("th", "TH")): String {
        return NumberFormat.getPercentInstance(locale).format(value)
    }
    
    fun formatDate(date: LocalDate, locale: Locale = Locale("th", "TH")): String {
        return DateTimeFormatter
            .ofLocalizedDate(FormatStyle.MEDIUM)
            .withLocale(locale)
            .format(date)
    }
}

// ============================================
// RTL Support
// ============================================

@Composable
fun ProductRow(product: Product) {
    // ✅ Use Start/End instead of Left/Right
    Row(
        modifier = Modifier.padding(
            start = 16.dp,    // Not paddingLeft
            end = 16.dp,      // Not paddingRight
            top = 8.dp,
            bottom = 8.dp
        )
    ) {
        AsyncImage(
            model = product.imageUrl,
            modifier = Modifier
                .align(Alignment.CenterVertically)
        )
        
        Spacer(Modifier.width(12.dp))
        
        Column {
            Text(product.name)
            // ✅ Text alignment handles RTL automatically
            Text(
                product.description,
                textAlign = TextAlign.Start  // Auto-mirrors for RTL
            )
        }
    }
}
```

---

## ขั้นตอนที่ 1953: Dark Mode Support

```kotlin
// ============================================
// Dynamic Color & Dark Mode
// ============================================

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
        darkTheme -> darkColorScheme(
            primary = Color(0xFF90CAF9),
            secondary = Color(0xFF80CBC4),
            background = Color(0xFF121212),
            surface = Color(0xFF1E1E1E)
        )
        else -> lightColorScheme(
            primary = Color(0xFF1565C0),
            secondary = Color(0xFF00695C),
            background = Color(0xFFFAFAFA),
            surface = Color.White
        )
    }
    
    MaterialTheme(
        colorScheme = colorScheme,
        typography = AppTypography,
        content = content
    )
}

// ============================================
// Theme Preview
// ============================================

@Preview(name = "Light Mode", uiMode = Configuration.UI_MODE_NIGHT_NO)
@Preview(name = "Dark Mode", uiMode = Configuration.UI_MODE_NIGHT_YES)
@Composable
fun ProductCardPreview() {
    AppTheme {
        ProductCard(
            product = previewProduct,
            onClick = {}
        )
    }
}

// ============================================
// Test Accessibility
// ============================================

@Test
fun productCard_accessibilityCheck() {
    composeTestRule.setContent {
        AppTheme {
            ProductCard(product = previewProduct, onClick = {})
        }
    }
    
    // Verify content descriptions exist
    composeTestRule
        .onNodeWithContentDescription("${previewProduct.name}, ราคา ฿35,000.00")
        .assertExists()
    
    // Verify touch target size
    composeTestRule
        .onNodeWithText("เพิ่มลงตะกร้า")
        .assertHeightIsAtLeast(48.dp)
    
    // Check color contrast (manual, or use AccessibilityChecks)
}
```

---

*Part 99 จบแล้ว | ก่อนหน้า: [Part 98](../part98/README.md) | ถัดไป: [Part 100](../part100/README.md)*
