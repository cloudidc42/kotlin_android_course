# Part 78: On-Device ML & TFLite
## ขั้นตอนที่ 1426-1450

---

## ขั้นตอนที่ 1426: On-Device ML Overview

```
On-Device Machine Learning:

Advantages:
- No network latency
- Works offline
- Privacy (data stays on device)
- No backend costs

Libraries:
1. TensorFlow Lite - custom models
2. ML Kit - pre-built models (Google)
3. ONNX Runtime - cross-platform models
4. MediaPipe - complex ML pipelines

Model Formats:
- .tflite - TensorFlow Lite
- .onnx - ONNX
- .mlmodel - CoreML (iOS only)

Optimization:
- Quantization (reduce model size)
- Hardware acceleration (GPU, NNAPI)
- Model pruning
```

---

## ขั้นตอนที่ 1427: TFLite Setup

```kotlin
// build.gradle.kts
// implementation("org.tensorflow:tensorflow-lite:2.x")
// implementation("org.tensorflow:tensorflow-lite-support:0.x")
// implementation("org.tensorflow:tensorflow-lite-metadata:0.x")
// implementation("org.tensorflow:tensorflow-lite-gpu:2.x")  // GPU acceleration

// Place model file in: app/src/main/assets/model.tflite

class ImageClassifier @Inject constructor(
    @ApplicationContext private val context: Context
) {
    
    private lateinit var interpreter: Interpreter
    private val labels = mutableListOf<String>()
    
    private val INPUT_SIZE = 224
    private val NUM_CLASSES = 1000
    
    fun initialize() {
        // Load model
        val modelFile = loadModelFile("mobilenet_v2.tflite")
        
        // Configure interpreter
        val options = Interpreter.Options().apply {
            numThreads = 4
            // Enable GPU acceleration
            addDelegate(GpuDelegate())
        }
        
        interpreter = Interpreter(modelFile, options)
        
        // Load labels
        context.assets.open("imagenet_labels.txt").bufferedReader().use {
            labels.addAll(it.readLines())
        }
    }
    
    private fun loadModelFile(filename: String): MappedByteBuffer {
        val fileDescriptor = context.assets.openFd(filename)
        val inputStream = FileInputStream(fileDescriptor.fileDescriptor)
        val fileChannel = inputStream.channel
        val startOffset = fileDescriptor.startOffset
        val declaredLength = fileDescriptor.declaredLength
        return fileChannel.map(FileChannel.MapMode.READ_ONLY, startOffset, declaredLength)
    }
    
    data class ClassificationResult(
        val label: String,
        val confidence: Float
    )
    
    fun classify(bitmap: Bitmap): List<ClassificationResult> {
        // Preprocess
        val resizedBitmap = Bitmap.createScaledBitmap(bitmap, INPUT_SIZE, INPUT_SIZE, true)
        
        val inputBuffer = TensorBuffer.createFixedSize(
            intArrayOf(1, INPUT_SIZE, INPUT_SIZE, 3),
            DataType.FLOAT32
        )
        
        // Normalize pixel values (0-255 → 0.0-1.0)
        val imageProcessor = ImageProcessor.Builder()
            .add(ResizeOp(INPUT_SIZE, INPUT_SIZE, ResizeOp.ResizeMethod.BILINEAR))
            .add(NormalizeOp(0f, 255f))
            .build()
        
        val tensorImage = TensorImage(DataType.FLOAT32)
        tensorImage.load(resizedBitmap)
        val processedImage = imageProcessor.process(tensorImage)
        
        // Run inference
        val outputBuffer = TensorBuffer.createFixedSize(
            intArrayOf(1, NUM_CLASSES),
            DataType.FLOAT32
        )
        
        interpreter.run(processedImage.buffer, outputBuffer.buffer.rewind())
        
        // Post-process: get top-5 results
        val scores = outputBuffer.floatArray
        
        return scores.mapIndexed { index, score ->
            ClassificationResult(labels.getOrElse(index) { "Unknown" }, score)
        }
            .sortedByDescending { it.confidence }
            .take(5)
    }
    
    fun close() {
        interpreter.close()
    }
}
```

---

## ขั้นตอนที่ 1428: Object Detection

```kotlin
// Object Detection ด้วย SSD MobileNet
class ObjectDetector @Inject constructor(
    @ApplicationContext private val context: Context
) {
    
    private lateinit var interpreter: Interpreter
    
    private val INPUT_SIZE = 300
    private val MAX_DETECTIONS = 10
    
    data class Detection(
        val label: String,
        val confidence: Float,
        val boundingBox: RectF  // normalized coordinates (0-1)
    )
    
    fun initialize() {
        val modelFile = loadModelFile("ssd_mobilenet_v2.tflite")
        interpreter = Interpreter(modelFile, Interpreter.Options().apply { numThreads = 4 })
    }
    
    fun detect(bitmap: Bitmap): List<Detection> {
        val resizedBitmap = Bitmap.createScaledBitmap(bitmap, INPUT_SIZE, INPUT_SIZE, true)
        
        // Prepare input: [1, 300, 300, 3] INT8
        val inputArray = Array(1) {
            Array(INPUT_SIZE) {
                Array(INPUT_SIZE) {
                    ByteArray(3)
                }
            }
        }
        
        for (y in 0 until INPUT_SIZE) {
            for (x in 0 until INPUT_SIZE) {
                val pixel = resizedBitmap.getPixel(x, y)
                inputArray[0][y][x][0] = ((pixel shr 16 and 0xFF) - 128).toByte()
                inputArray[0][y][x][1] = ((pixel shr 8 and 0xFF) - 128).toByte()
                inputArray[0][y][x][2] = ((pixel and 0xFF) - 128).toByte()
            }
        }
        
        // Output arrays
        val outputLocations = Array(1) { Array(MAX_DETECTIONS) { FloatArray(4) } }
        val outputClasses = Array(1) { FloatArray(MAX_DETECTIONS) }
        val outputScores = Array(1) { FloatArray(MAX_DETECTIONS) }
        val numDetections = FloatArray(1)
        
        val outputMap = mapOf(
            0 to outputLocations,
            1 to outputClasses,
            2 to outputScores,
            3 to numDetections
        )
        
        interpreter.runForMultipleInputsOutputs(arrayOf(inputArray), outputMap)
        
        val detections = mutableListOf<Detection>()
        val count = numDetections[0].toInt().coerceAtMost(MAX_DETECTIONS)
        
        for (i in 0 until count) {
            val score = outputScores[0][i]
            if (score < 0.5f) continue
            
            val box = outputLocations[0][i]
            // [top, left, bottom, right]
            val rect = RectF(box[1], box[0], box[3], box[2])
            
            detections.add(
                Detection(
                    label = labels[outputClasses[0][i].toInt()],
                    confidence = score,
                    boundingBox = rect
                )
            )
        }
        
        return detections
    }
}
```

---

## ขั้นตอนที่ 1429: Drawing Detection Results

```kotlin
// ============================================
// Canvas overlay สำหรับ drawing bounding boxes
// ============================================

@Composable
fun DetectionOverlay(
    detections: List<ObjectDetector.Detection>,
    imageWidth: Int,
    imageHeight: Int,
    modifier: Modifier = Modifier
) {
    Canvas(modifier = modifier) {
        val scaleX = size.width / imageWidth
        val scaleY = size.height / imageHeight
        
        detections.forEach { detection ->
            val rect = detection.boundingBox
            
            // Scale to canvas coordinates
            val left = rect.left * imageWidth * scaleX
            val top = rect.top * imageHeight * scaleY
            val right = rect.right * imageWidth * scaleX
            val bottom = rect.bottom * imageHeight * scaleY
            
            // Draw bounding box
            drawRect(
                color = Color.Red,
                topLeft = Offset(left, top),
                size = Size(right - left, bottom - top),
                style = Stroke(width = 3f)
            )
            
            // Draw label
            drawContext.canvas.nativeCanvas.drawText(
                "${detection.label} ${(detection.confidence * 100).toInt()}%",
                left,
                top - 10,
                android.graphics.Paint().apply {
                    color = android.graphics.Color.RED
                    textSize = 40f
                    style = android.graphics.Paint.Style.FILL
                }
            )
        }
    }
}

// Real-time detection with CameraX
val imageAnalyzer = ImageAnalysis.Builder()
    .setTargetResolution(Size(640, 480))
    .setBackpressureStrategy(ImageAnalysis.STRATEGY_KEEP_ONLY_LATEST)
    .build()
    .also {
        it.setAnalyzer(executor) { imageProxy ->
            val bitmap = imageProxy.toBitmap()
            val detections = objectDetector.detect(bitmap)
            
            withContext(Dispatchers.Main) {
                viewModel.updateDetections(detections, bitmap.width, bitmap.height)
            }
            
            imageProxy.close()
        }
    }
```

---

## ขั้นตอนที่ 1430: ML Kit Language Detection

```kotlin
// ML Kit Language ID - detect language ของ text
// implementation("com.google.mlkit:language-id:17.x")
// implementation("com.google.mlkit:translate:17.x")

class LanguageProcessor @Inject constructor() {
    
    private val languageIdentifier = LanguageIdentification.getClient(
        LanguageIdentificationOptions.Builder()
            .setConfidenceThreshold(0.5f)
            .build()
    )
    
    private val translators = mutableMapOf<Pair<String, String>, Translator>()
    
    suspend fun detectLanguage(text: String): String {
        return suspendCancellableCoroutine { continuation ->
            languageIdentifier.identifyLanguage(text)
                .addOnSuccessListener { languageCode ->
                    continuation.resume(languageCode)  // "th", "en", "ja", etc.
                }
                .addOnFailureListener { continuation.resumeWithException(it) }
        }
    }
    
    suspend fun translate(text: String, sourceLanguage: String, targetLanguage: String): String {
        val key = Pair(sourceLanguage, targetLanguage)
        
        val translator = translators.getOrPut(key) {
            Translation.getClient(
                TranslatorOptions.Builder()
                    .setSourceLanguage(sourceLanguage)
                    .setTargetLanguage(targetLanguage)
                    .build()
            )
        }
        
        // Download model if needed
        translator.downloadModelIfNeeded().await()
        
        return suspendCancellableCoroutine { continuation ->
            translator.translate(text)
                .addOnSuccessListener { continuation.resume(it) }
                .addOnFailureListener { continuation.resumeWithException(it) }
        }
    }
    
    fun close() {
        translators.values.forEach { it.close() }
        languageIdentifier.close()
    }
}
```

---

## แบบฝึกหัด Part 78

```kotlin
// แบบฝึกหัด: Smart Receipt Scanner

// สร้าง ReceiptScannerViewModel ที่:
// 1. รับ Bitmap จาก CameraX
// 2. ใช้ ML Kit OCR อ่านข้อความ
// 3. Parse ข้อความหาราคา (regex: ฿?\d+\.?\d*)
// 4. Parse วันที่ (regex: \d{1,2}/\d{1,2}/\d{2,4})
// 5. สรุปเป็น ReceiptData
// 6. บันทึกลง Room

data class ReceiptData(
    val merchantName: String?,
    val date: LocalDate?,
    val items: List<ReceiptItem>,
    val total: Double?,
    val rawText: String
)

data class ReceiptItem(
    val name: String,
    val price: Double
)

@HiltViewModel
class ReceiptScannerViewModel @Inject constructor(
    private val textRecognizer: TextRecognizer,
    private val receiptRepository: ReceiptRepository
) : ViewModel() {
    
    fun processReceipt(bitmap: Bitmap) {
        viewModelScope.launch {
            // TODO: implement OCR + parsing
        }
    }
}
```

---

*Part 78 จบแล้ว | ก่อนหน้า: [Part 77](../part77/README.md) | ถัดไป: [Part 79](../part79/README.md)*
