# Part 59: Custom Composables & Modifiers
## ขั้นตอนที่ 951-975

---

## ขั้นตอนที่ 951: Custom Modifier

```kotlin
// ============================================
// สร้าง Custom Modifier
// ============================================

// 1. Modifier.composed - สำหรับ stateful modifiers
fun Modifier.coloredBorder(
    color: Color,
    strokeWidth: Dp = 2.dp,
    cornerRadius: Dp = 8.dp
): Modifier = this.then(
    Modifier.drawBehind {
        val strokePx = strokeWidth.toPx()
        val radius = cornerRadius.toPx()
        drawRoundRect(
            color = color,
            style = Stroke(width = strokePx),
            cornerRadius = CornerRadius(radius)
        )
    }
)

// 2. Custom Modifier ที่ใช้ DrawModifier
fun Modifier.badge(count: Int): Modifier = composed {
    if (count <= 0) return@composed this
    
    val badgeColor = MaterialTheme.colorScheme.error
    val textColor = MaterialTheme.colorScheme.onError
    val textSize = 10.sp
    val textPaint = remember {
        android.graphics.Paint().apply {
            color = android.graphics.Color.WHITE
            textAlign = android.graphics.Paint.Align.CENTER
            textSize = textSize.toPx()
        }
    }
    
    this.drawWithContent {
        drawContent()
        
        val badgeRadius = 12.dp.toPx()
        val x = size.width - badgeRadius * 0.5f
        val y = badgeRadius * 0.5f
        
        drawCircle(
            color = badgeColor,
            radius = badgeRadius,
            center = Offset(x, y)
        )
        
        drawContext.canvas.nativeCanvas.drawText(
            if (count > 99) "99+" else count.toString(),
            x, y + textSize.toPx() / 3,
            textPaint
        )
    }
}

// 3. Modifier Extension - reusable patterns
fun Modifier.shimmerBackground(isLoading: Boolean): Modifier = composed {
    if (!isLoading) return@composed this
    
    val infiniteTransition = rememberInfiniteTransition(label = "shimmer")
    val offset by infiniteTransition.animateFloat(
        initialValue = -1f,
        targetValue = 2f,
        animationSpec = infiniteRepeatable(tween(1200, easing = LinearEasing)),
        label = "offset"
    )
    
    val gradient = Brush.linearGradient(
        0.0f to Color(0xFFE0E0E0),
        0.5f to Color(0xFFF5F5F5),
        1.0f to Color(0xFFE0E0E0),
        start = Offset(offset * 1000f, 0f),
        end = Offset((offset + 1f) * 1000f, 0f)
    )
    
    this.background(gradient)
}

// ใช้งาน Custom Modifiers
@Composable
fun ProductCard(product: Product, isLoading: Boolean) {
    Card(
        modifier = Modifier
            .fillMaxWidth()
            .padding(8.dp)
    ) {
        Box {
            Column(
                modifier = Modifier
                    .padding(16.dp)
                    .shimmerBackground(isLoading)
            ) {
                // content
            }
            
            Icon(
                Icons.Default.Notifications,
                contentDescription = null,
                modifier = Modifier
                    .align(Alignment.TopEnd)
                    .badge(5)
                    .padding(8.dp)
            )
        }
    }
}
```

---

## ขั้นตอนที่ 952: Custom Layout

```kotlin
// ============================================
// Custom Layout - เหมือน ViewGroup แบบกำหนดเอง
// ============================================

// FlowRow - Wrap items ไปบรรทัดถัดไปอัตโนมัติ
@Composable
fun FlowRow(
    modifier: Modifier = Modifier,
    spacing: Dp = 8.dp,
    content: @Composable () -> Unit
) {
    Layout(
        modifier = modifier,
        content = content
    ) { measurables, constraints ->
        val spacingPx = spacing.roundToPx()
        var currentX = 0
        var currentY = 0
        var rowHeight = 0
        
        val placeables = measurables.map { measurable ->
            measurable.measure(constraints.copy(minWidth = 0))
        }
        
        // คำนวณตำแหน่ง
        val positions = placeables.map { placeable ->
            if (currentX + placeable.width > constraints.maxWidth && currentX > 0) {
                // ขึ้นบรรทัดใหม่
                currentX = 0
                currentY += rowHeight + spacingPx
                rowHeight = 0
            }
            
            val x = currentX
            val y = currentY
            currentX += placeable.width + spacingPx
            rowHeight = maxOf(rowHeight, placeable.height)
            
            Offset(x.toFloat(), y.toFloat())
        }
        
        val totalHeight = currentY + rowHeight
        
        layout(constraints.maxWidth, totalHeight) {
            placeables.forEachIndexed { i, placeable ->
                placeable.placeRelative(
                    x = positions[i].x.toInt(),
                    y = positions[i].y.toInt()
                )
            }
        }
    }
}

// ใช้งาน FlowRow
@Composable
fun TagsRow(tags: List<String>) {
    FlowRow(spacing = 8.dp) {
        tags.forEach { tag ->
            SuggestionChip(
                onClick = {},
                label = { Text(tag) }
            )
        }
    }
}
```

---

## ขั้นตอนที่ 953: Canvas Drawing

```kotlin
// ============================================
// Canvas - วาดกราฟิกแบบ custom
// ============================================

@Composable
fun DonutChart(
    data: List<Pair<String, Float>>,  // label to percentage
    modifier: Modifier = Modifier
) {
    val colors = listOf(
        Color(0xFF6C5CE7),
        Color(0xFF00B894),
        Color(0xFFE17055),
        Color(0xFF74B9FF),
        Color(0xFFFD79A8)
    )
    
    Canvas(modifier = modifier.size(200.dp)) {
        val strokeWidth = 40.dp.toPx()
        val radius = (size.minDimension - strokeWidth) / 2
        val topLeft = Offset(
            (size.width - radius * 2) / 2,
            (size.height - radius * 2) / 2
        )
        
        var startAngle = -90f
        val total = data.sumOf { it.second.toDouble() }.toFloat()
        
        data.forEachIndexed { index, (_, value) ->
            val sweepAngle = (value / total) * 360f
            
            drawArc(
                color = colors[index % colors.size],
                startAngle = startAngle,
                sweepAngle = sweepAngle - 2f,  // gap
                useCenter = false,
                style = Stroke(width = strokeWidth, cap = StrokeCap.Round),
                topLeft = topLeft,
                size = Size(radius * 2, radius * 2)
            )
            
            startAngle += sweepAngle
        }
    }
}

// ============================================
// Progress Bar แบบ custom
// ============================================

@Composable
fun GradientProgressBar(
    progress: Float,  // 0f - 1f
    modifier: Modifier = Modifier
) {
    val animatedProgress by animateFloatAsState(
        targetValue = progress,
        animationSpec = tween(1000, easing = EaseOutCubic),
        label = "progress"
    )
    
    Canvas(modifier = modifier.height(12.dp).fillMaxWidth()) {
        val cornerRadius = size.height / 2
        
        // Background
        drawRoundRect(
            color = Color(0xFFE0E0E0),
            cornerRadius = CornerRadius(cornerRadius)
        )
        
        // Progress with gradient
        if (animatedProgress > 0f) {
            drawRoundRect(
                brush = Brush.horizontalGradient(
                    colors = listOf(Color(0xFF6C5CE7), Color(0xFFA855F7))
                ),
                size = Size(size.width * animatedProgress, size.height),
                cornerRadius = CornerRadius(cornerRadius)
            )
        }
    }
}
```

---

## ขั้นตอนที่ 954: Custom Text

```kotlin
// ============================================
// AnnotatedString - styled text
// ============================================

@Composable
fun HighlightedText(text: String, keyword: String) {
    val annotatedString = buildAnnotatedString {
        var lastIndex = 0
        val regex = Regex(keyword, RegexOption.IGNORE_CASE)
        
        regex.findAll(text).forEach { match ->
            // Normal text before match
            append(text.substring(lastIndex, match.range.first))
            
            // Highlighted match
            withStyle(
                style = SpanStyle(
                    background = Color.Yellow,
                    fontWeight = FontWeight.Bold,
                    color = Color.Black
                )
            ) {
                append(match.value)
            }
            
            lastIndex = match.range.last + 1
        }
        
        // Remaining text
        append(text.substring(lastIndex))
    }
    
    Text(annotatedString)
}

// ClickableText ด้วย annotations
@Composable
fun TextWithLinks(text: String, links: Map<String, String>) {
    val annotatedString = buildAnnotatedString {
        var currentIndex = 0
        
        links.forEach { (linkText, url) ->
            val startIndex = text.indexOf(linkText, currentIndex)
            if (startIndex != -1) {
                append(text.substring(currentIndex, startIndex))
                
                pushStringAnnotation("URL", url)
                withStyle(SpanStyle(
                    color = MaterialTheme.colorScheme.primary,
                    textDecoration = TextDecoration.Underline
                )) {
                    append(linkText)
                }
                pop()
                
                currentIndex = startIndex + linkText.length
            }
        }
        append(text.substring(currentIndex))
    }
    
    val uriHandler = LocalUriHandler.current
    
    ClickableText(
        text = annotatedString,
        onClick = { offset ->
            annotatedString.getStringAnnotations("URL", offset, offset)
                .firstOrNull()?.let { annotation ->
                    uriHandler.openUri(annotation.item)
                }
        }
    )
}
```

---

## แบบฝึกหัด Part 59

```kotlin
// แบบฝึกหัด: สร้าง Star Rating Component

// ต้องการผลลัพธ์:
// ★★★★☆  (4/5 stars)
// - สามารถ animate ได้
// - รองรับ half-star
// - มี accessibility support

@Composable
fun StarRating(
    rating: Float,  // 0.0 - 5.0
    maxRating: Int = 5,
    onRatingChange: ((Float) -> Unit)? = null,  // null = read-only
    modifier: Modifier = Modifier
) {
    // TODO: implement star rating
    // 1. วาด filled stars ด้วย Canvas
    // 2. วาด half-filled stars
    // 3. วาด empty stars
    // 4. ถ้า onRatingChange != null ให้ clickable
    // 5. เพิ่ม animation เมื่อ rating เปลี่ยน
    // 6. เพิ่ม semantics: stateDescription = "$rating out of $maxRating stars"
    
    Row(
        modifier = modifier.semantics {
            stateDescription = "$rating จาก $maxRating ดาว"
        }
    ) {
        repeat(maxRating) { index ->
            val starFill = (rating - index).coerceIn(0f, 1f)
            StarIcon(
                fill = starFill,
                onClick = onRatingChange?.let {
                    { it(index + 1f) }
                }
            )
        }
    }
}

@Composable
fun StarIcon(fill: Float, onClick: (() -> Unit)?) {
    // TODO: วาด star ด้วย Canvas
    // fill = 0f → ว่าง, 0.5f → ครึ่ง, 1f → เต็ม
}
```

---

*Part 59 จบแล้ว | ก่อนหน้า: [Part 58](../part58/README.md) | ถัดไป: [Part 60](../part60/README.md)*
