# Part 68: CameraX & Media
## ขั้นตอนที่ 1176-1200

---

## ขั้นตอนที่ 1176: CameraX Basics

```kotlin
// build.gradle.kts
// implementation("androidx.camera:camera-core:1.4.x")
// implementation("androidx.camera:camera-camera2:1.4.x")
// implementation("androidx.camera:camera-lifecycle:1.4.x")
// implementation("androidx.camera:camera-view:1.4.x")
// implementation("androidx.camera:camera-extensions:1.4.x")

// Permissions ใน AndroidManifest.xml
// <uses-permission android:name="android.permission.CAMERA"/>
// <uses-permission android:name="android.permission.RECORD_AUDIO"/>

// ============================================
// Camera Use Cases
// ============================================

// 1. Preview - แสดงกล้องบนหน้าจอ
// 2. ImageCapture - ถ่ายรูป
// 3. ImageAnalysis - วิเคราะห์ frame realtime
// 4. VideoCapture - บันทึกวิดีโอ

@AndroidEntryPoint
class CameraFragment : Fragment() {
    
    private var imageCapture: ImageCapture? = null
    private var cameraProvider: ProcessCameraProvider? = null
    private lateinit var cameraExecutor: ExecutorService
    
    override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
        super.onViewCreated(view, savedInstanceState)
        
        cameraExecutor = Executors.newSingleThreadExecutor()
        
        startCamera()
    }
    
    private fun startCamera() {
        val cameraProviderFuture = ProcessCameraProvider.getInstance(requireContext())
        
        cameraProviderFuture.addListener({
            cameraProvider = cameraProviderFuture.get()
            bindCameraUseCases()
        }, ContextCompat.getMainExecutor(requireContext()))
    }
    
    private fun bindCameraUseCases() {
        val cameraProvider = cameraProvider ?: return
        
        val preview = Preview.Builder().build().also { preview ->
            preview.setSurfaceProvider(binding.viewFinder.surfaceProvider)
        }
        
        imageCapture = ImageCapture.Builder()
            .setCaptureMode(ImageCapture.CAPTURE_MODE_MINIMIZE_LATENCY)
            .build()
        
        val imageAnalyzer = ImageAnalysis.Builder()
            .setBackpressureStrategy(ImageAnalysis.STRATEGY_KEEP_ONLY_LATEST)
            .build()
            .also { analysis ->
                analysis.setAnalyzer(cameraExecutor) { imageProxy ->
                    analyzeImage(imageProxy)
                }
            }
        
        try {
            cameraProvider.unbindAll()
            cameraProvider.bindToLifecycle(
                viewLifecycleOwner,
                CameraSelector.DEFAULT_BACK_CAMERA,
                preview,
                imageCapture,
                imageAnalyzer
            )
        } catch (e: Exception) {
            Log.e(TAG, "Use case binding failed", e)
        }
    }
    
    fun takePhoto() {
        val imageCapture = imageCapture ?: return
        
        val outputFile = File(
            requireContext().getExternalFilesDir(Environment.DIRECTORY_PICTURES),
            "photo_${System.currentTimeMillis()}.jpg"
        )
        
        val outputOptions = ImageCapture.OutputFileOptions.Builder(outputFile).build()
        
        imageCapture.takePicture(
            outputOptions,
            ContextCompat.getMainExecutor(requireContext()),
            object : ImageCapture.OnImageSavedCallback {
                override fun onImageSaved(output: ImageCapture.OutputFileResults) {
                    val savedUri = output.savedUri ?: Uri.fromFile(outputFile)
                    onPhotoSaved(savedUri)
                }
                
                override fun onError(exception: ImageCaptureException) {
                    Log.e(TAG, "Photo capture failed: ${exception.message}", exception)
                }
            }
        )
    }
    
    private fun analyzeImage(imageProxy: ImageProxy) {
        // Process image here
        // Remember to close when done
        imageProxy.close()
    }
    
    override fun onDestroy() {
        super.onDestroy()
        cameraExecutor.shutdown()
    }
}
```

---

## ขั้นตอนที่ 1177: CameraX ใน Compose

```kotlin
// ============================================
// CameraX Preview ใน Compose
// ============================================

@Composable
fun CameraPreview(
    onImageCaptured: (Uri) -> Unit,
    modifier: Modifier = Modifier
) {
    val context = LocalContext.current
    val lifecycleOwner = LocalLifecycleOwner.current
    
    val preview = remember { Preview.Builder().build() }
    val imageCapture = remember {
        ImageCapture.Builder()
            .setCaptureMode(ImageCapture.CAPTURE_MODE_MINIMIZE_LATENCY)
            .build()
    }
    
    var isFrontCamera by remember { mutableStateOf(false) }
    
    LaunchedEffect(isFrontCamera) {
        val cameraProvider = ProcessCameraProvider.getInstance(context).await()
        
        val cameraSelector = if (isFrontCamera) {
            CameraSelector.DEFAULT_FRONT_CAMERA
        } else {
            CameraSelector.DEFAULT_BACK_CAMERA
        }
        
        cameraProvider.unbindAll()
        cameraProvider.bindToLifecycle(
            lifecycleOwner,
            cameraSelector,
            preview,
            imageCapture
        )
    }
    
    Box(modifier = modifier) {
        // Camera preview
        AndroidView(
            factory = { ctx ->
                PreviewView(ctx).also { preview.setSurfaceProvider(it.surfaceProvider) }
            },
            modifier = Modifier.fillMaxSize()
        )
        
        // Controls
        Row(
            modifier = Modifier
                .fillMaxWidth()
                .align(Alignment.BottomCenter)
                .padding(bottom = 32.dp),
            horizontalArrangement = Arrangement.SpaceEvenly,
            verticalAlignment = Alignment.CenterVertically
        ) {
            // Flip camera button
            IconButton(onClick = { isFrontCamera = !isFrontCamera }) {
                Icon(Icons.Default.FlipCameraAndroid, "Flip Camera", tint = Color.White)
            }
            
            // Capture button
            val captureScope = rememberCoroutineScope()
            Box(
                modifier = Modifier
                    .size(72.dp)
                    .clip(CircleShape)
                    .background(Color.White)
                    .clickable {
                        captureScope.launch {
                            capturePhoto(context, imageCapture, onImageCaptured)
                        }
                    }
            )
            
            Spacer(Modifier.size(48.dp))
        }
    }
}

suspend fun capturePhoto(
    context: Context,
    imageCapture: ImageCapture,
    onCaptured: (Uri) -> Unit
) {
    val file = File(
        context.getExternalFilesDir(Environment.DIRECTORY_PICTURES),
        "photo_${System.currentTimeMillis()}.jpg"
    )
    
    val output = ImageCapture.OutputFileOptions.Builder(file).build()
    
    suspendCancellableCoroutine { continuation ->
        imageCapture.takePicture(
            output,
            ContextCompat.getMainExecutor(context),
            object : ImageCapture.OnImageSavedCallback {
                override fun onImageSaved(result: ImageCapture.OutputFileResults) {
                    val uri = result.savedUri ?: Uri.fromFile(file)
                    continuation.resume(uri)
                    onCaptured(uri)
                }
                
                override fun onError(exception: ImageCaptureException) {
                    continuation.resumeWithException(exception)
                }
            }
        )
    }
}
```

---

## ขั้นตอนที่ 1178: ML Kit Integration

```kotlin
// ============================================
// ML Kit - Text Recognition
// ============================================

// implementation("com.google.mlkit:text-recognition:16.x")

class TextRecognizer @Inject constructor() {
    
    private val recognizer = TextRecognition.getClient(TextRecognizerOptions.DEFAULT_OPTIONS)
    
    suspend fun recognizeText(bitmap: Bitmap): String {
        return suspendCancellableCoroutine { continuation ->
            val image = InputImage.fromBitmap(bitmap, 0)
            
            recognizer.process(image)
                .addOnSuccessListener { visionText ->
                    continuation.resume(visionText.text)
                }
                .addOnFailureListener { exception ->
                    continuation.resumeWithException(exception)
                }
        }
    }
    
    fun recognizeTextFromImageProxy(imageProxy: ImageProxy): Task<Text> {
        val inputImage = InputImage.fromMediaImage(
            imageProxy.image!!,
            imageProxy.imageInfo.rotationDegrees
        )
        return recognizer.process(inputImage)
    }
}

// ============================================
// ML Kit - Face Detection
// ============================================

// implementation("com.google.mlkit:face-detection:16.x")

class FaceDetector @Inject constructor() {
    
    private val detector = FaceDetection.getClient(
        FaceDetectorOptions.Builder()
            .setPerformanceMode(FaceDetectorOptions.PERFORMANCE_MODE_FAST)
            .setLandmarkMode(FaceDetectorOptions.LANDMARK_MODE_ALL)
            .setClassificationMode(FaceDetectorOptions.CLASSIFICATION_MODE_ALL)
            .enableTracking()
            .build()
    )
    
    data class FaceResult(
        val trackingId: Int?,
        val bounds: android.graphics.Rect,
        val smileProb: Float?,
        val leftEyeOpenProb: Float?,
        val rightEyeOpenProb: Float?
    )
    
    suspend fun detectFaces(bitmap: Bitmap): List<FaceResult> {
        return suspendCancellableCoroutine { continuation ->
            val image = InputImage.fromBitmap(bitmap, 0)
            
            detector.process(image)
                .addOnSuccessListener { faces ->
                    val results = faces.map { face ->
                        FaceResult(
                            trackingId = face.trackingId,
                            bounds = face.boundingBox,
                            smileProb = face.smilingProbability,
                            leftEyeOpenProb = face.leftEyeOpenProbability,
                            rightEyeOpenProb = face.rightEyeOpenProbability
                        )
                    }
                    continuation.resume(results)
                }
                .addOnFailureListener { continuation.resumeWithException(it) }
        }
    }
}

// ============================================
// Image Capture with Analysis Pipeline
// ============================================

val imageAnalyzer = ImageAnalysis.Builder()
    .setBackpressureStrategy(ImageAnalysis.STRATEGY_KEEP_ONLY_LATEST)
    .setOutputImageFormat(ImageAnalysis.OUTPUT_IMAGE_FORMAT_YUV_420_888)
    .build()

imageAnalyzer.setAnalyzer(cameraExecutor) { imageProxy ->
    val bitmap = imageProxy.toBitmap()
    
    // Run ML analysis
    CoroutineScope(Dispatchers.Default).launch {
        val text = textRecognizer.recognizeText(bitmap)
        val faces = faceDetector.detectFaces(bitmap)
        
        withContext(Dispatchers.Main) {
            // Update UI with results
        }
    }
    
    imageProxy.close()
}
```

---

## แบบฝึกหัด Part 68

```kotlin
// แบบฝึกหัด: QR Code Scanner ด้วย CameraX + ML Kit

// implementation("com.google.mlkit:barcode-scanning:17.x")

class QrCodeScanner @Inject constructor() {
    
    private val scanner = BarcodeScanning.getClient(
        BarcodeScannerOptions.Builder()
            .setBarcodeFormats(Barcode.FORMAT_QR_CODE)
            .build()
    )
    
    // TODO: implement scan function
    fun scan(imageProxy: ImageProxy): Task<List<Barcode>> {
        val inputImage = InputImage.fromMediaImage(
            imageProxy.image!!,
            imageProxy.imageInfo.rotationDegrees
        )
        return scanner.process(inputImage)
    }
}

// TODO: สร้าง QrScannerScreen ที่:
// 1. แสดง Camera preview
// 2. วิเคราะห์ frame ด้วย ML Kit
// 3. เมื่อเจอ QR code → หยุด scan, แสดงผล
// 4. มีปุ่ม "scan again"
// 5. กล่อง focus ตรงกลางหน้าจอ
// 6. Vibrate เมื่อ scan สำเร็จ

@Composable
fun QrScannerScreen(
    onQrDetected: (String) -> Unit,
    modifier: Modifier = Modifier
) {
    // TODO: implement
}
```

---

*Part 68 จบแล้ว | ก่อนหน้า: [Part 67](../part67/README.md) | ถัดไป: [Part 69](../part69/README.md)*
