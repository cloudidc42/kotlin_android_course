# Part 79: ARCore & Augmented Reality
## ขั้นตอนที่ 1451-1475

---

## ขั้นตอนที่ 1451: ARCore Overview

```
ARCore - Google's AR platform for Android:

Features:
1. Motion Tracking - ติดตาม position/orientation
2. Environmental Understanding - detect surfaces (planes)
3. Light Estimation - ประเมินแสงในสภาพแวดล้อม
4. Augmented Faces - overlay content บนใบหน้า
5. Depth API - depth map จาก camera

Requirements:
- Device ต้องรองรับ ARCore
- ARCore Services ต้อง install
- Camera permission

Use Cases:
- Furniture placement (IKEA)
- Navigation overlay
- Face filters
- Gaming (Pokémon GO)
- Education (3D models)
```

---

## ขั้นตอนที่ 1452: ARCore + Sceneform Setup

```kotlin
// build.gradle.kts
// implementation("io.github.sceneview:arsceneview:2.x")
// Sceneform is deprecated - use SceneView as alternative

// AndroidManifest.xml
/*
<uses-permission android:name="android.permission.CAMERA"/>
<uses-feature android:name="android.hardware.camera.ar" android:required="true"/>

<application>
    <meta-data
        android:name="com.google.ar.core"
        android:value="required"/>
</application>
*/

// Check ARCore availability
fun checkArCoreAvailability(context: Context): ArCoreStatus {
    return when (ArCoreApk.getInstance().checkAvailability(context)) {
        ArCoreApk.Availability.SUPPORTED_INSTALLED -> ArCoreStatus.AVAILABLE
        ArCoreApk.Availability.SUPPORTED_APK_TOO_OLD,
        ArCoreApk.Availability.SUPPORTED_NOT_INSTALLED -> ArCoreStatus.NEEDS_INSTALL
        else -> ArCoreStatus.NOT_SUPPORTED
    }
}

enum class ArCoreStatus { AVAILABLE, NEEDS_INSTALL, NOT_SUPPORTED }
```

---

## ขั้นตอนที่ 1453: AR Scene ใน Compose

```kotlin
// ============================================
// AR View ด้วย SceneView
// ============================================

@Composable
fun ArFurniturePlacer(
    models: List<FurnitureModel>,
    modifier: Modifier = Modifier
) {
    val context = LocalContext.current
    var selectedModel by remember { mutableStateOf<FurnitureModel?>(null) }
    var placedObjects by remember { mutableStateOf<List<ArNode>>(emptyList()) }
    
    Box(modifier = modifier) {
        // AR Camera View
        AndroidView(
            factory = { ctx ->
                ArSceneView(ctx).apply {
                    // Configure AR scene
                    planeRenderer.isEnabled = true
                    planeRenderer.isVisible = true
                    
                    // Handle tap to place object
                    setOnTapArPlaneListener { hitResult, plane, motionEvent ->
                        selectedModel?.let { model ->
                            placeObject(hitResult, model)
                        }
                    }
                }
            },
            modifier = Modifier.fillMaxSize()
        )
        
        // UI Overlay
        Column(
            modifier = Modifier
                .align(Alignment.BottomCenter)
                .padding(16.dp)
        ) {
            // Model selector
            LazyRow(horizontalArrangement = Arrangement.spacedBy(8.dp)) {
                items(models) { model ->
                    FurnitureChip(
                        model = model,
                        selected = selectedModel == model,
                        onClick = { selectedModel = model }
                    )
                }
            }
        }
        
        // Instructions
        if (selectedModel != null) {
            Text(
                text = "แตะบนพื้นเพื่อวาง ${selectedModel!!.name}",
                modifier = Modifier
                    .align(Alignment.TopCenter)
                    .padding(top = 32.dp)
                    .background(Color.Black.copy(alpha = 0.5f))
                    .padding(horizontal = 16.dp, vertical = 8.dp),
                color = Color.White
            )
        }
    }
}

data class FurnitureModel(
    val id: Long,
    val name: String,
    val modelUri: String,  // path to .glb file
    val thumbnailUrl: String,
    val widthMeters: Float,
    val heightMeters: Float
)
```

---

## ขั้นตอนที่ 1454: AR Session Management

```kotlin
class ArSessionManager @Inject constructor(
    @ApplicationContext private val context: Context
) {
    
    private var session: Session? = null
    
    fun createSession(): Session? {
        return try {
            val session = Session(context)
            
            val config = Config(session).apply {
                // Enable plane detection
                planeFindingMode = Config.PlaneFindingMode.HORIZONTAL_AND_VERTICAL
                
                // Enable light estimation
                lightEstimationMode = Config.LightEstimationMode.ENVIRONMENTAL_HDR
                
                // Update mode
                updateMode = Config.UpdateMode.LATEST_CAMERA_IMAGE
            }
            
            session.configure(config)
            this.session = session
            session
        } catch (e: Exception) {
            null
        }
    }
    
    fun getTrackingState(): TrackingState? {
        return session?.update()?.camera?.trackingState
    }
    
    fun getDetectedPlanes(): List<Plane> {
        return session?.getAllTrackables(Plane::class.java)?.toList() ?: emptyList()
    }
    
    fun performHitTest(x: Float, y: Float): HitResult? {
        val frame = session?.update() ?: return null
        return frame.hitTest(x, y).firstOrNull { hit ->
            val trackable = hit.trackable
            trackable is Plane && trackable.isPoseInPolygon(hit.hitPose)
        }
    }
    
    fun close() {
        session?.close()
        session = null
    }
}

// Anchor management
class AnchorManager {
    
    private val anchors = mutableMapOf<Long, Anchor>()
    private var nextId = 0L
    
    fun createAnchor(hitResult: HitResult): Long {
        val anchor = hitResult.createAnchor()
        val id = nextId++
        anchors[id] = anchor
        return id
    }
    
    fun getAnchor(id: Long): Anchor? = anchors[id]
    
    fun removeAnchor(id: Long) {
        anchors.remove(id)?.detach()
    }
    
    fun clearAll() {
        anchors.values.forEach { it.detach() }
        anchors.clear()
    }
}
```

---

## ขั้นตอนที่ 1455: 3D Model Loading

```kotlin
// ============================================
// Load .glb Model ด้วย ModelRenderable
// ============================================

class ModelLoader @Inject constructor(
    @ApplicationContext private val context: Context
) {
    
    private val modelCache = mutableMapOf<String, ModelRenderable>()
    
    suspend fun loadModel(modelPath: String): ModelRenderable? {
        modelCache[modelPath]?.let { return it }
        
        return suspendCancellableCoroutine { continuation ->
            ModelRenderable.builder()
                .setSource(context, Uri.parse(modelPath))
                .setIsFilamentGltf(true)
                .build()
                .thenAccept { renderable ->
                    modelCache[modelPath] = renderable
                    continuation.resume(renderable)
                }
                .exceptionally { exception ->
                    continuation.resumeWithException(exception)
                    null
                }
        }
    }
    
    fun clearCache() {
        modelCache.clear()
    }
}

// Place model at AR hit point
fun placeObject(
    scene: Scene,
    hitResult: HitResult,
    renderable: ModelRenderable
): AnchorNode {
    val anchor = hitResult.createAnchor()
    
    val anchorNode = AnchorNode(anchor).apply {
        setParent(scene)
    }
    
    val transformableNode = TransformableNode(scene.session!!.sharedCamera).apply {
        setParent(anchorNode)
        this.renderable = renderable
        select()
    }
    
    return anchorNode
}
```

---

## แบบฝึกหัด Part 79

```kotlin
// แบบฝึกหัด: AR Product Viewer

// สร้าง AR Product Viewer ที่:
// 1. Load product 3D models (.glb) จาก network
// 2. แสดง AR preview บนพื้น
// 3. Scale/rotate model ด้วย gesture
// 4. Screenshot พร้อม AR overlay
// 5. แสดง product info card ถัดจาก model

data class Product3D(
    val id: Long,
    val name: String,
    val description: String,
    val price: Double,
    val modelUrl: String,
    val thumbnailUrl: String
)

@HiltViewModel
class ArProductViewModel @Inject constructor(
    private val productRepository: ProductRepository,
    private val modelLoader: ModelLoader
) : ViewModel() {
    
    private val _selectedProduct = MutableStateFlow<Product3D?>(null)
    val selectedProduct: StateFlow<Product3D?> = _selectedProduct.asStateFlow()
    
    private val _loadedModel = MutableStateFlow<ModelRenderable?>(null)
    val loadedModel: StateFlow<ModelRenderable?> = _loadedModel.asStateFlow()
    
    fun selectProduct(product: Product3D) {
        _selectedProduct.value = product
        viewModelScope.launch {
            _loadedModel.value = modelLoader.loadModel(product.modelUrl)
        }
    }
}
```

---

*Part 79 จบแล้ว | ก่อนหน้า: [Part 78](../part78/README.md) | ถัดไป: [Part 80](../part80/README.md)*
