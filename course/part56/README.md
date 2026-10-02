# Part 56: Compose Animations
## ขั้นตอนที่ 876-900

---

## ขั้นตอนที่ 876: Animation Basics

```kotlin
// ============================================
// animateAsState - animate single value
// ============================================

@Composable
fun AnimatedCounter(targetValue: Int) {
    val animatedValue by animateIntAsState(
        targetValue = targetValue,
        animationSpec = tween(durationMillis = 500, easing = EaseOutBounce),
        label = "counter"
    )
    Text("$animatedValue", style = MaterialTheme.typography.displayLarge)
}

// animate Float
@Composable
fun FadeButton(visible: Boolean, onClick: () -> Unit) {
    val alpha by animateFloatAsState(
        targetValue = if (visible) 1f else 0f,
        animationSpec = tween(300),
        label = "alpha"
    )
    Button(
        onClick = onClick,
        modifier = Modifier.alpha(alpha)
    ) { Text("Click Me") }
}

// animate Color
@Composable
fun ThemeToggle(isDark: Boolean) {
    val backgroundColor by animateColorAsState(
        targetValue = if (isDark) Color(0xFF1A1A2E) else Color(0xFFF5F5F5),
        animationSpec = tween(500),
        label = "background"
    )
    val textColor by animateColorAsState(
        targetValue = if (isDark) Color.White else Color.Black,
        label = "text"
    )
    
    Box(
        modifier = Modifier
            .fillMaxSize()
            .background(backgroundColor),
        contentAlignment = Alignment.Center
    ) {
        Text("Hello", color = textColor)
    }
}

// animate Dp
@Composable
fun ExpandableCard(expanded: Boolean) {
    val cardHeight by animateDpAsState(
        targetValue = if (expanded) 200.dp else 80.dp,
        animationSpec = spring(dampingRatio = Spring.DampingRatioMediumBouncy),
        label = "height"
    )
    
    Card(modifier = Modifier.fillMaxWidth().height(cardHeight)) {
        // content
    }
}
```

---

## ขั้นตอนที่ 877: AnimationSpec

```kotlin
// ============================================
// ชนิดของ AnimationSpec
// ============================================

// 1. tween - เส้นตรงพร้อม easing
val tweenSpec = tween<Float>(
    durationMillis = 500,
    delayMillis = 100,
    easing = FastOutSlowInEasing  // Material Design easing
)

// Custom easing
val customEasing = CubicBezierEasing(0.4f, 0.0f, 0.2f, 1.0f)

// 2. spring - ฟิสิกส์จริง
val springSpec = spring<Float>(
    dampingRatio = Spring.DampingRatioLowBouncy,  // มีการ bounce
    stiffness = Spring.StiffnessMedium
)

// DampingRatio options:
// - NoBouncy (1.0)
// - LowBouncy (0.75)
// - MediumBouncy (0.5)
// - HighBouncy (0.2)

// Stiffness options:
// - High (10_000)
// - Medium (1_500)
// - MediumLow (400)
// - Low (200)
// - VeryLow (50)

// 3. keyframes - กำหนด keyframe แต่ละช่วง
val keyframesSpec = keyframes<Float> {
    durationMillis = 1000
    0f at 0 with FastOutSlowInEasing
    0.8f at 500 with LinearEasing
    1.0f at 1000
}

// 4. repeatable - วนซ้ำ
val repeatableSpec = infiniteRepeatable<Float>(
    animation = tween(1000),
    repeatMode = RepeatMode.Reverse
)

// 5. snap - ไม่มี animation
val snapSpec = snap<Float>(delayMillis = 0)
```

---

## ขั้นตอนที่ 878: Animate Visibility และ Content

```kotlin
// ============================================
// AnimatedVisibility
// ============================================

@Composable
fun ToggleableContent() {
    var visible by remember { mutableStateOf(true) }
    
    Column {
        Button(onClick = { visible = !visible }) {
            Text(if (visible) "ซ่อน" else "แสดง")
        }
        
        AnimatedVisibility(
            visible = visible,
            enter = fadeIn() + expandVertically(),
            exit = fadeOut() + shrinkVertically()
        ) {
            Card(modifier = Modifier.padding(16.dp)) {
                Text("เนื้อหาที่ซ่อนได้", modifier = Modifier.padding(16.dp))
            }
        }
    }
}

// Custom enter/exit
AnimatedVisibility(
    visible = showDialog,
    enter = slideInVertically(
        initialOffsetY = { it },  // มาจากด้านล่าง
        animationSpec = spring(dampingRatio = Spring.DampingRatioMediumBouncy)
    ),
    exit = slideOutVertically(targetOffsetY = { it })
) {
    BottomSheetContent()
}

// ============================================
// AnimatedContent - transition ระหว่าง content
// ============================================

@Composable
fun CounterWithAnimation(count: Int) {
    AnimatedContent(
        targetState = count,
        transitionSpec = {
            if (targetState > initialState) {
                slideInVertically { -it } + fadeIn() togetherWith
                slideOutVertically { it } + fadeOut()
            } else {
                slideInVertically { it } + fadeIn() togetherWith
                slideOutVertically { -it } + fadeOut()
            } using SizeTransform(clip = false)
        },
        label = "counter"
    ) { targetCount ->
        Text("$targetCount", style = MaterialTheme.typography.displayMedium)
    }
}

// Tab content transition
@Composable
fun TabbedContent(selectedTab: Int) {
    AnimatedContent(
        targetState = selectedTab,
        transitionSpec = {
            if (targetState > initialState) {
                slideInHorizontally { it } + fadeIn() togetherWith
                slideOutHorizontally { -it } + fadeOut()
            } else {
                slideInHorizontally { -it } + fadeIn() togetherWith
                slideOutHorizontally { it } + fadeOut()
            }
        },
        label = "tab_content"
    ) { tab ->
        when (tab) {
            0 -> HomeContent()
            1 -> ProfileContent()
            2 -> SettingsContent()
        }
    }
}
```

---

## ขั้นตอนที่ 879: Transition

```kotlin
// ============================================
// updateTransition - หลาย animation พร้อมกัน
// ============================================

enum class BoxState { Small, Large }

@Composable
fun AnimatedBox() {
    var state by remember { mutableStateOf(BoxState.Small) }
    val transition = updateTransition(targetState = state, label = "box")
    
    val size by transition.animateDp(label = "size") { boxState ->
        if (boxState == BoxState.Small) 64.dp else 200.dp
    }
    
    val color by transition.animateColor(label = "color") { boxState ->
        if (boxState == BoxState.Small) MaterialTheme.colorScheme.primary
        else MaterialTheme.colorScheme.tertiary
    }
    
    val cornerRadius by transition.animateDp(label = "corners") { boxState ->
        if (boxState == BoxState.Small) 8.dp else 50.dp
    }
    
    Box(
        modifier = Modifier
            .size(size)
            .clip(RoundedCornerShape(cornerRadius))
            .background(color)
            .clickable { state = if (state == BoxState.Small) BoxState.Large else BoxState.Small }
    )
}

// ============================================
// Crossfade - เปลี่ยน content แบบ fade
// ============================================

@Composable
fun ScreenWithCrossfade(route: String) {
    Crossfade(
        targetState = route,
        animationSpec = tween(300),
        label = "screen"
    ) { currentRoute ->
        when (currentRoute) {
            "home" -> HomeScreen()
            "profile" -> ProfileScreen()
            else -> NotFoundScreen()
        }
    }
}
```

---

## ขั้นตอนที่ 880: Infinite Animation

```kotlin
// ============================================
// Infinite Animations - loading, pulsing
// ============================================

@Composable
fun LoadingDots() {
    val infiniteTransition = rememberInfiniteTransition(label = "loading")
    
    Row(
        horizontalArrangement = Arrangement.spacedBy(8.dp),
        verticalAlignment = Alignment.CenterVertically
    ) {
        repeat(3) { index ->
            val alpha by infiniteTransition.animateFloat(
                initialValue = 0.2f,
                targetValue = 1.0f,
                animationSpec = infiniteRepeatable(
                    animation = tween(600),
                    repeatMode = RepeatMode.Reverse,
                    initialStartOffset = StartOffset(index * 200)  // เหลื่อมกัน
                ),
                label = "dot_$index"
            )
            
            Box(
                modifier = Modifier
                    .size(10.dp)
                    .clip(CircleShape)
                    .background(MaterialTheme.colorScheme.primary.copy(alpha = alpha))
            )
        }
    }
}

// Shimmer Loading Effect
@Composable
fun ShimmerEffect(modifier: Modifier = Modifier) {
    val infiniteTransition = rememberInfiniteTransition(label = "shimmer")
    val shimmerX by infiniteTransition.animateFloat(
        initialValue = -1000f,
        targetValue = 1000f,
        animationSpec = infiniteRepeatable(tween(1200, easing = LinearEasing)),
        label = "shimmer_x"
    )
    
    val brush = Brush.linearGradient(
        colors = listOf(
            Color(0xFFE0E0E0),
            Color(0xFFF5F5F5),
            Color(0xFFE0E0E0)
        ),
        start = Offset(shimmerX - 200f, 0f),
        end = Offset(shimmerX + 200f, 0f)
    )
    
    Box(modifier = modifier.background(brush))
}

// Pulsing Heart
@Composable
fun PulsingFavorite(isFavorite: Boolean) {
    val infiniteTransition = rememberInfiniteTransition(label = "pulse")
    val scale by infiniteTransition.animateFloat(
        initialValue = 1f,
        targetValue = 1.2f,
        animationSpec = infiniteRepeatable(
            animation = tween(500, easing = FastOutSlowInEasing),
            repeatMode = RepeatMode.Reverse
        ),
        label = "scale"
    )
    
    Icon(
        imageVector = if (isFavorite) Icons.Filled.Favorite else Icons.Outlined.FavoriteBorder,
        contentDescription = null,
        tint = if (isFavorite) Color.Red else Color.Gray,
        modifier = Modifier
            .scale(if (isFavorite) scale else 1f)
            .size(28.dp)
    )
}
```

---

## แบบฝึกหัด Part 56

```kotlin
// แบบฝึกหัด: สร้าง Animated Onboarding Screen

// TODO: สร้าง Onboarding ที่มี:
// 1. Pager ที่ swipe ระหว่าง 3 หน้าได้
// 2. Animated dots indicator ที่ขยาย/หดตาม page ปัจจุบัน
// 3. Slide transition ระหว่าง page
// 4. Fade + slide in สำหรับ content แต่ละ page
// 5. Floating action button ที่ animate จาก "ถัดไป" เป็น "เริ่มต้น"

data class OnboardingPage(
    val title: String,
    val description: String,
    val imageRes: Int
)

@Composable
fun OnboardingScreen(
    pages: List<OnboardingPage>,
    onFinish: () -> Unit
) {
    // TODO: implement with HorizontalPager + animations
}

// Animated dots indicator
@Composable
fun PageIndicator(pageCount: Int, currentPage: Int) {
    Row(horizontalArrangement = Arrangement.spacedBy(8.dp)) {
        repeat(pageCount) { index ->
            val width by animateDpAsState(
                targetValue = if (index == currentPage) 24.dp else 8.dp,
                label = "dot_width"
            )
            val color by animateColorAsState(
                targetValue = if (index == currentPage)
                    MaterialTheme.colorScheme.primary
                else
                    MaterialTheme.colorScheme.primary.copy(alpha = 0.3f),
                label = "dot_color"
            )
            Box(
                modifier = Modifier
                    .height(8.dp)
                    .width(width)
                    .clip(CircleShape)
                    .background(color)
            )
        }
    }
}
```

---

*Part 56 จบแล้ว | ก่อนหน้า: [Part 55](../part55/README.md) | ถัดไป: [Part 57](../part57/README.md)*
