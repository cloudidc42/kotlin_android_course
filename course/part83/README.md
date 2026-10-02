# Part 83: Advanced Compose Animation
## ขั้นตอนที่ 1551-1575

---

## ขั้นตอนที่ 1551: Animated Navigation Transitions

```kotlin
// ============================================
// Custom Navigation Transitions ด้วย Compose Navigation
// ============================================

// implementation("androidx.navigation:navigation-compose:2.x")
// implementation("com.google.accompanist:accompanist-navigation-animation:0.x")

@Composable
fun AnimatedNavHost(
    navController: NavHostController,
    startDestination: String,
    modifier: Modifier = Modifier
) {
    AnimatedNavHost(
        navController = navController,
        startDestination = startDestination,
        modifier = modifier,
        // Default transitions
        enterTransition = {
            slideInHorizontally(
                initialOffsetX = { fullWidth -> fullWidth },
                animationSpec = tween(300, easing = FastOutSlowInEasing)
            ) + fadeIn(animationSpec = tween(300))
        },
        exitTransition = {
            slideOutHorizontally(
                targetOffsetX = { fullWidth -> -fullWidth / 3 },
                animationSpec = tween(300)
            ) + fadeOut(animationSpec = tween(300))
        },
        popEnterTransition = {
            slideInHorizontally(
                initialOffsetX = { fullWidth -> -fullWidth / 3 },
                animationSpec = tween(300)
            ) + fadeIn(animationSpec = tween(300))
        },
        popExitTransition = {
            slideOutHorizontally(
                targetOffsetX = { fullWidth -> fullWidth },
                animationSpec = tween(300)
            ) + fadeOut(animationSpec = tween(300))
        }
    ) {
        composable("home") { HomeScreen() }
        composable("detail/{id}",
            enterTransition = {
                // Custom transition for this destination
                scaleIn(
                    initialScale = 0.9f,
                    animationSpec = tween(300)
                ) + fadeIn(animationSpec = tween(300))
            },
            exitTransition = {
                scaleOut(
                    targetScale = 0.9f,
                    animationSpec = tween(300)
                ) + fadeOut(animationSpec = tween(300))
            }
        ) { backStackEntry ->
            val id = backStackEntry.arguments?.getString("id")
            DetailScreen(id = id ?: "")
        }
    }
}
```

---

## ขั้นตอนที่ 1552: Shared Element Transition

```kotlin
// ============================================
// Shared Element Transition (Android 5.0+, Compose 1.7+)
// ============================================

@OptIn(ExperimentalSharedTransitionApi::class)
@Composable
fun SharedElementDemo() {
    SharedTransitionLayout {
        var showDetail by remember { mutableStateOf(false) }
        
        AnimatedContent(
            targetState = showDetail,
            transitionSpec = {
                fadeIn() togetherWith fadeOut()
            }
        ) { isDetail ->
            if (!isDetail) {
                // List view
                LazyColumn {
                    items(products) { product ->
                        ProductListItem(
                            product = product,
                            onClick = { showDetail = true },
                            sharedTransitionScope = this@SharedTransitionLayout,
                            animatedVisibilityScope = this@AnimatedContent
                        )
                    }
                }
            } else {
                // Detail view
                ProductDetail(
                    product = selectedProduct,
                    onBack = { showDetail = false },
                    sharedTransitionScope = this@SharedTransitionLayout,
                    animatedVisibilityScope = this@AnimatedContent
                )
            }
        }
    }
}

@OptIn(ExperimentalSharedTransitionApi::class)
@Composable
fun ProductListItem(
    product: Product,
    onClick: () -> Unit,
    sharedTransitionScope: SharedTransitionScope,
    animatedVisibilityScope: AnimatedVisibilityScope
) {
    with(sharedTransitionScope) {
        Row(
            modifier = Modifier
                .fillMaxWidth()
                .clickable(onClick = onClick)
                .padding(16.dp)
        ) {
            AsyncImage(
                model = product.imageUrl,
                contentDescription = null,
                modifier = Modifier
                    .size(80.dp)
                    .sharedElement(
                        state = rememberSharedContentState(key = "image_${product.id}"),
                        animatedVisibilityScope = animatedVisibilityScope
                    )
                    .clip(RoundedCornerShape(8.dp))
            )
            
            Spacer(Modifier.width(16.dp))
            
            Text(
                text = product.name,
                modifier = Modifier.sharedElement(
                    state = rememberSharedContentState(key = "title_${product.id}"),
                    animatedVisibilityScope = animatedVisibilityScope
                ),
                style = MaterialTheme.typography.titleMedium
            )
        }
    }
}
```

---

## ขั้นตอนที่ 1553: Physics-Based Animation

```kotlin
// ============================================
// Spring Animation (มีความรู้สึก natural)
// ============================================

@Composable
fun PhysicsAnimations() {
    
    // Springy card flip
    var isFlipped by remember { mutableStateOf(false) }
    val rotation by animateFloatAsState(
        targetValue = if (isFlipped) 180f else 0f,
        animationSpec = spring(
            dampingRatio = Spring.DampingRatioMediumBouncy,
            stiffness = Spring.StiffnessLow
        )
    )
    
    Box(
        modifier = Modifier
            .size(200.dp)
            .graphicsLayer { rotationY = rotation }
            .clickable { isFlipped = !isFlipped }
    ) {
        if (rotation < 90f) {
            CardFront()
        } else {
            CardBack(modifier = Modifier.graphicsLayer { rotationY = 180f })
        }
    }
    
    // Elastic scale on press
    var isPressed by remember { mutableStateOf(false) }
    val scale by animateFloatAsState(
        targetValue = if (isPressed) 0.9f else 1f,
        animationSpec = spring(
            dampingRatio = Spring.DampingRatioHighBouncy,
            stiffness = Spring.StiffnessMedium
        )
    )
    
    Box(
        modifier = Modifier
            .scale(scale)
            .size(100.dp)
            .background(MaterialTheme.colorScheme.primary, CircleShape)
            .pointerInput(Unit) {
                detectTapGestures(
                    onPress = {
                        isPressed = true
                        tryAwaitRelease()
                        isPressed = false
                    }
                )
            }
    )
}

// ============================================
// Velocity-based fling animation
// ============================================

@Composable
fun SwipeableDismissCard(
    onDismiss: () -> Unit
) {
    val offsetX = remember { Animatable(0f) }
    val scope = rememberCoroutineScope()
    
    Box(
        modifier = Modifier
            .offset { IntOffset(offsetX.value.roundToInt(), 0) }
            .pointerInput(Unit) {
                detectHorizontalDragGestures(
                    onDragEnd = {
                        scope.launch {
                            if (abs(offsetX.value) > size.width / 3) {
                                // Fling off screen
                                offsetX.animateTo(
                                    targetValue = if (offsetX.value > 0) size.width.toFloat() else -size.width.toFloat(),
                                    animationSpec = tween(200)
                                )
                                onDismiss()
                            } else {
                                // Snap back
                                offsetX.animateTo(0f, spring())
                            }
                        }
                    },
                    onHorizontalDrag = { _, dragAmount ->
                        scope.launch { offsetX.snapTo(offsetX.value + dragAmount) }
                    }
                )
            }
    ) {
        // Card content
    }
}
```

---

## ขั้นตอนที่ 1554: Animated Gradient & Particle Effects

```kotlin
// ============================================
// Animated Gradient Background
// ============================================

@Composable
fun AnimatedGradientBackground(
    modifier: Modifier = Modifier,
    content: @Composable BoxScope.() -> Unit
) {
    val infiniteTransition = rememberInfiniteTransition()
    
    val color1 by infiniteTransition.animateColor(
        initialValue = Color(0xFF6C5CE7),
        targetValue = Color(0xFFE17055),
        animationSpec = infiniteRepeatable(
            animation = tween(3000, easing = LinearEasing),
            repeatMode = RepeatMode.Reverse
        )
    )
    
    val color2 by infiniteTransition.animateColor(
        initialValue = Color(0xFF00B894),
        targetValue = Color(0xFF0984E3),
        animationSpec = infiniteRepeatable(
            animation = tween(4000, easing = LinearEasing),
            repeatMode = RepeatMode.Reverse
        )
    )
    
    Box(
        modifier = modifier.background(
            Brush.linearGradient(listOf(color1, color2))
        ),
        content = content
    )
}

// ============================================
// Floating Particles
// ============================================

data class Particle(
    val id: Int,
    val x: Float,
    val y: Float,
    val radius: Float,
    val speedX: Float,
    val speedY: Float,
    val color: Color,
    val alpha: Float
)

@Composable
fun ParticleBackground(
    particleCount: Int = 50,
    modifier: Modifier = Modifier
) {
    val particles = remember {
        List(particleCount) {
            Particle(
                id = it,
                x = Random.nextFloat(),
                y = Random.nextFloat(),
                radius = Random.nextFloat() * 6f + 2f,
                speedX = (Random.nextFloat() - 0.5f) * 0.002f,
                speedY = -(Random.nextFloat() * 0.003f + 0.001f),
                color = Color(
                    red = Random.nextFloat(),
                    green = Random.nextFloat(),
                    blue = Random.nextFloat()
                ),
                alpha = Random.nextFloat() * 0.5f + 0.3f
            )
        }.toMutableList()
    }
    
    var tick by remember { mutableLongStateOf(0L) }
    
    LaunchedEffect(Unit) {
        while (true) {
            withFrameMillis { frameTime ->
                tick = frameTime
                // Update particle positions
            }
        }
    }
    
    Canvas(modifier = modifier) {
        particles.forEach { particle ->
            // Update position
            val newX = (particle.x + particle.speedX + 1f) % 1f
            val newY = if (particle.y - particle.speedY < 0) 1f else particle.y - particle.speedY
            
            drawCircle(
                color = particle.color.copy(alpha = particle.alpha),
                radius = particle.radius,
                center = Offset(newX * size.width, newY * size.height)
            )
        }
    }
}
```

---

## แบบฝึกหัด Part 83

```kotlin
// แบบฝึกหัด: Animated Onboarding Screen

// สร้าง OnboardingScreen ที่:
// 1. 3 หน้า slide ด้วย HorizontalPager
// 2. Shared element transition ของ illustration
// 3. Animated text reveal (ทีละตัวอักษร)
// 4. Progress indicator ที่ animate
// 5. Particle background ที่ animate ต่อเนื่อง
// 6. "Get Started" button ที่มี spring animation

@Composable
fun OnboardingScreen(
    pages: List<OnboardingPage>,
    onComplete: () -> Unit
) {
    val pagerState = rememberPagerState { pages.size }
    
    // TODO: implement animated onboarding
    HorizontalPager(
        state = pagerState,
        modifier = Modifier.fillMaxSize()
    ) { pageIndex ->
        OnboardingPage(page = pages[pageIndex])
    }
}

data class OnboardingPage(
    val title: String,
    val description: String,
    val imageRes: Int,
    val backgroundColor: Color
)
```

---

*Part 83 จบแล้ว | ก่อนหน้า: [Part 82](../part82/README.md) | ถัดไป: [Part 84](../part84/README.md)*
