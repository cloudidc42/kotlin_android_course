# Part 37: Compose - Animation พื้นฐาน
## ขั้นตอนที่ 701-725

---

## ขั้นตอนที่ 701: animateFloatAsState และ animateColorAsState

Animation ใน Compose ทำได้ง่ายมาก ด้วย animate*AsState - state เปลี่ยน → animation เริ่มทำงานเอง

```kotlin
import androidx.compose.animation.animateColorAsState
import androidx.compose.animation.core.*
import androidx.compose.foundation.background
import androidx.compose.foundation.clickable
import androidx.compose.foundation.layout.*
import androidx.compose.material3.*
import androidx.compose.runtime.*
import androidx.compose.ui.Modifier
import androidx.compose.ui.draw.alpha
import androidx.compose.ui.graphics.Color
import androidx.compose.ui.unit.dp

@Composable
fun AnimateFloatExample() {
    var isExpanded by remember { mutableStateOf(false) }

    // animateFloatAsState - animate ค่า Float
    val height by animateFloatAsState(
        targetValue = if (isExpanded) 200f else 100f,
        animationSpec = tween(durationMillis = 500, easing = FastOutSlowInEasing),
        label = "height animation"
    )

    val alpha by animateFloatAsState(
        targetValue = if (isExpanded) 1f else 0.3f,
        animationSpec = tween(300),
        label = "alpha animation"
    )

    // animateColorAsState - animate สี
    val backgroundColor by animateColorAsState(
        targetValue = if (isExpanded) Color(0xFF4CAF50) else Color(0xFF2196F3),
        animationSpec = tween(400),
        label = "color animation"
    )

    Column(modifier = Modifier.padding(16.dp)) {
        Box(
            modifier = Modifier
                .fillMaxWidth()
                .height(height.dp)
                .background(backgroundColor)
                .clickable { isExpanded = !isExpanded },
            contentAlignment = Alignment.Center
        ) {
            Text(
                text = if (isExpanded) "คลิกเพื่อย่อ" else "คลิกเพื่อขยาย",
                color = Color.White,
                modifier = Modifier.alpha(alpha)
            )
        }
    }
}

@Composable
fun AnimateDpExample() {
    var selected by remember { mutableStateOf(false) }

    // animateDpAsState
    val elevation by animateDpAsState(
        targetValue = if (selected) 8.dp else 2.dp,
        label = "elevation"
    )

    val padding by animateDpAsState(
        targetValue = if (selected) 24.dp else 8.dp,
        animationSpec = spring(dampingRatio = Spring.DampingRatioMediumBouncy),
        label = "padding"
    )

    Card(
        elevation = CardDefaults.cardElevation(defaultElevation = elevation),
        modifier = Modifier
            .fillMaxWidth()
            .padding(padding)
            .clickable { selected = !selected }
    ) {
        Text(
            "คลิกเพื่อเลือก",
            modifier = Modifier.padding(16.dp)
        )
    }
}
```

---

## ขั้นตอนที่ 702: AnimatedVisibility

AnimatedVisibility ทำให้ component ปรากฏ/หายด้วย animation

```kotlin
import androidx.compose.animation.*
import androidx.compose.animation.core.tween
import androidx.compose.foundation.layout.*
import androidx.compose.material3.*
import androidx.compose.runtime.*
import androidx.compose.ui.Modifier
import androidx.compose.ui.unit.dp

@Composable
fun AnimatedVisibilityExamples() {
    var showContent by remember { mutableStateOf(false) }
    var showMessage by remember { mutableStateOf(false) }

    Column(
        modifier = Modifier.padding(16.dp),
        verticalArrangement = Arrangement.spacedBy(8.dp)
    ) {
        Button(onClick = { showContent = !showContent }) {
            Text(if (showContent) "ซ่อน" else "แสดง")
        }

        // AnimatedVisibility พื้นฐาน
        AnimatedVisibility(visible = showContent) {
            Card(modifier = Modifier.fillMaxWidth()) {
                Text("เนื้อหาที่ซ่อน/แสดง", modifier = Modifier.padding(16.dp))
            }
        }

        Divider()

        Button(onClick = { showMessage = !showMessage }) {
            Text("Toggle Message")
        }

        // Custom Enter/Exit Animation
        AnimatedVisibility(
            visible = showMessage,
            enter = slideInVertically(initialOffsetY = { -it }) + fadeIn(tween(300)),
            exit = slideOutVertically(targetOffsetY = { -it }) + fadeOut(tween(300))
        ) {
            Card(
                colors = CardDefaults.cardColors(
                    containerColor = MaterialTheme.colorScheme.primaryContainer
                )
            ) {
                Row(
                    modifier = Modifier.padding(16.dp),
                    horizontalArrangement = Arrangement.spacedBy(8.dp)
                ) {
                    Icon(Icons.Default.CheckCircle, null)
                    Text("บันทึกสำเร็จ!")
                }
            }
        }

        // Expand/Collapse (slide in/out vertical)
        var expanded by remember { mutableStateOf(false) }
        OutlinedButton(onClick = { expanded = !expanded }) {
            Text(if (expanded) "ย่อ ▲" else "ขยาย ▼")
        }
        AnimatedVisibility(
            visible = expanded,
            enter = expandVertically() + fadeIn(),
            exit = shrinkVertically() + fadeOut()
        ) {
            Column(modifier = Modifier.padding(8.dp)) {
                repeat(5) { i ->
                    Text("รายละเอียด ${i + 1}")
                }
            }
        }
    }
}
```

---

## ขั้นตอนที่ 703: Crossfade และ AnimatedContent

```kotlin
import androidx.compose.animation.*
import androidx.compose.animation.core.tween
import androidx.compose.foundation.layout.*
import androidx.compose.material3.*
import androidx.compose.runtime.*
import androidx.compose.ui.Modifier
import androidx.compose.ui.unit.dp

@Composable
fun CrossfadeExample() {
    var currentTab by remember { mutableStateOf("home") }

    Column {
        // Tab selector
        Row(horizontalArrangement = Arrangement.spacedBy(8.dp)) {
            listOf("home", "search", "profile").forEach { tab ->
                Button(
                    onClick = { currentTab = tab },
                    colors = ButtonDefaults.buttonColors(
                        containerColor = if (currentTab == tab)
                            MaterialTheme.colorScheme.primary
                        else
                            MaterialTheme.colorScheme.surfaceVariant
                    )
                ) {
                    Text(tab)
                }
            }
        }

        Spacer(Modifier.height(16.dp))

        // Crossfade - สลับ content ด้วย fade
        Crossfade(
            targetState = currentTab,
            animationSpec = tween(300),
            label = "tab crossfade"
        ) { tab ->
            Box(
                modifier = Modifier
                    .fillMaxWidth()
                    .height(200.dp),
                contentAlignment = Alignment.Center
            ) {
                Text("หน้า: $tab", style = MaterialTheme.typography.headlineMedium)
            }
        }
    }
}

@Composable
fun AnimatedContentExample() {
    var count by remember { mutableStateOf(0) }

    Column(horizontalAlignment = Alignment.CenterHorizontally) {
        // AnimatedContent - animation เมื่อ content เปลี่ยน
        AnimatedContent(
            targetState = count,
            transitionSpec = {
                // ตัวเลขขึ้น: slide ขึ้น, ตัวเลขลง: slide ลง
                if (targetState > initialState) {
                    slideInVertically { -it } + fadeIn() togetherWith
                        slideOutVertically { it } + fadeOut()
                } else {
                    slideInVertically { it } + fadeIn() togetherWith
                        slideOutVertically { -it } + fadeOut()
                }
            },
            label = "count animation"
        ) { targetCount ->
            Text(
                text = "$targetCount",
                style = MaterialTheme.typography.displayLarge
            )
        }

        Row(horizontalArrangement = Arrangement.spacedBy(8.dp)) {
            Button(onClick = { count-- }) { Text("−") }
            Button(onClick = { count++ }) { Text("+") }
        }
    }
}
```

---

## ขั้นตอนที่ 704: Infinite Animation

```kotlin
import androidx.compose.animation.core.*
import androidx.compose.foundation.Canvas
import androidx.compose.foundation.layout.*
import androidx.compose.material3.*
import androidx.compose.runtime.*
import androidx.compose.ui.Modifier
import androidx.compose.ui.draw.rotate
import androidx.compose.ui.graphics.Color
import androidx.compose.ui.unit.dp

@Composable
fun InfiniteAnimationExamples() {
    // rememberInfiniteTransition - loop animation ไม่หยุด
    val infiniteTransition = rememberInfiniteTransition(label = "infinite")

    // หมุนตลอดเวลา
    val rotation by infiniteTransition.animateFloat(
        initialValue = 0f,
        targetValue = 360f,
        animationSpec = infiniteRepeatable(
            animation = tween(2000, easing = LinearEasing),
            repeatMode = RepeatMode.Restart
        ),
        label = "rotation"
    )

    // กระพริบ
    val alpha by infiniteTransition.animateFloat(
        initialValue = 0.2f,
        targetValue = 1f,
        animationSpec = infiniteRepeatable(
            animation = tween(800),
            repeatMode = RepeatMode.Reverse
        ),
        label = "alpha"
    )

    // สีที่เปลี่ยนไปมา
    val color by infiniteTransition.animateColor(
        initialValue = Color.Red,
        targetValue = Color.Blue,
        animationSpec = infiniteRepeatable(
            animation = tween(1500),
            repeatMode = RepeatMode.Reverse
        ),
        label = "color"
    )

    Column(
        horizontalAlignment = Alignment.CenterHorizontally,
        verticalArrangement = Arrangement.spacedBy(24.dp),
        modifier = Modifier.padding(16.dp)
    ) {
        // Loading spinner
        CircularProgressIndicator(
            modifier = Modifier
                .size(48.dp)
                .rotate(rotation)
        )

        // กระพริบ
        Text(
            "กำลังโหลด...",
            modifier = Modifier.alpha(alpha),
            style = MaterialTheme.typography.bodyLarge
        )

        // สีเปลี่ยน
        Canvas(modifier = Modifier.size(100.dp)) {
            drawCircle(color = color, radius = size.minDimension / 2)
        }
    }
}

// Shimmer Effect (Loading Placeholder)
@Composable
fun ShimmerEffect() {
    val shimmerTranslate by rememberInfiniteTransition(label = "shimmer")
        .animateFloat(
            initialValue = 0f,
            targetValue = 1000f,
            animationSpec = infiniteRepeatable(
                animation = tween(1200, easing = LinearEasing),
                repeatMode = RepeatMode.Restart
            ),
            label = "shimmer translate"
        )

    Column(verticalArrangement = Arrangement.spacedBy(8.dp)) {
        repeat(5) {
            Box(
                modifier = Modifier
                    .fillMaxWidth()
                    .height(60.dp)
                    .background(
                        brush = Brush.horizontalGradient(
                            colors = listOf(
                                Color.LightGray.copy(alpha = 0.6f),
                                Color.LightGray.copy(alpha = 0.2f),
                                Color.LightGray.copy(alpha = 0.6f)
                            ),
                            startX = shimmerTranslate - 200f,
                            endX = shimmerTranslate + 200f
                        ),
                        shape = RoundedCornerShape(8.dp)
                    )
            )
        }
    }
}
```

---

## ขั้นตอนที่ 705: Spring และ Keyframe Animation

```kotlin
import androidx.compose.animation.core.*
import androidx.compose.foundation.clickable
import androidx.compose.foundation.layout.*
import androidx.compose.material3.*
import androidx.compose.runtime.*
import androidx.compose.ui.Modifier
import androidx.compose.ui.unit.dp

@Composable
fun SpringAnimationExample() {
    var pressed by remember { mutableStateOf(false) }

    // Spring - animation ที่มีการสั่น (bouncy)
    val scale by animateFloatAsState(
        targetValue = if (pressed) 0.8f else 1f,
        animationSpec = spring(
            dampingRatio = Spring.DampingRatioMediumBouncy, // ความสั่น
            stiffness = Spring.StiffnessLow // ความแข็ง
        ),
        label = "scale"
    )

    Box(
        modifier = Modifier
            .size(100.dp)
            .scale(scale)
            .background(MaterialTheme.colorScheme.primary, CircleShape)
            .clickable { pressed = !pressed },
        contentAlignment = Alignment.Center
    ) {
        Text("กด!", color = MaterialTheme.colorScheme.onPrimary)
    }
}

@Composable
fun KeyframeAnimationExample() {
    var triggered by remember { mutableStateOf(false) }

    val offsetY by animateFloatAsState(
        targetValue = if (triggered) 0f else 0f,
        animationSpec = keyframes {
            durationMillis = 1000
            0f at 0 with LinearEasing
            -100f at 250 with FastOutSlowInEasing
            0f at 500 with LinearOutSlowInEasing
            -50f at 750
            0f at 1000
        },
        label = "bounce"
    )

    Column(horizontalAlignment = Alignment.CenterHorizontally) {
        Text(
            "🎈",
            style = MaterialTheme.typography.displayLarge,
            modifier = Modifier.offset(y = offsetY.dp)
        )
        Button(onClick = { triggered = !triggered }) {
            Text("Bounce!")
        }
    }
}
```

---

*Part 37 จบแล้ว | ก่อนหน้า: [Part 36](../part36/README.md) | ถัดไป: [Part 38](../part38/README.md)*
