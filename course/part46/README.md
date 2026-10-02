# Part 46: Android - Coil Image Loading
## ขั้นตอนที่ 926-950

---

## ขั้นตอนที่ 926: ติดตั้งและใช้งาน Coil

Coil (Coroutine Image Loader) คือ library โหลดรูปภาพสำหรับ Android ที่รองรับ Compose

```kotlin
// build.gradle.kts
dependencies {
    implementation("io.coil-kt:coil-compose:2.5.0")
    // GIF support
    implementation("io.coil-kt:coil-gif:2.5.0")
    // SVG support
    implementation("io.coil-kt:coil-svg:2.5.0")
    // Video frame support
    implementation("io.coil-kt:coil-video:2.5.0")
}

// AndroidManifest.xml
// <uses-permission android:name="android.permission.INTERNET" />
```

```kotlin
import coil.compose.AsyncImage
import coil.compose.SubcomposeAsyncImage
import coil.request.ImageRequest
import androidx.compose.ui.platform.LocalContext

@Composable
fun BasicCoilImage() {
    // AsyncImage - วิธีง่ายที่สุด
    AsyncImage(
        model = "https://picsum.photos/400/300",
        contentDescription = "รูปสุ่ม",
        modifier = Modifier
            .fillMaxWidth()
            .height(200.dp),
        contentScale = ContentScale.Crop
    )
}
```

---

## ขั้นตอนที่ 927: AsyncImage พร้อม Placeholder และ Error

```kotlin
import coil.compose.AsyncImage
import coil.compose.AsyncImagePainter
import coil.compose.SubcomposeAsyncImage
import coil.request.ImageRequest

@Composable
fun CoilImageWithPlaceholder() {
    val context = LocalContext.current

    // วิธีที่ 1: ใช้ placeholder/error parameter
    AsyncImage(
        model = ImageRequest.Builder(context)
            .data("https://picsum.photos/400/300")
            .crossfade(true)           // fade transition
            .crossfade(500)            // 500ms fade
            .build(),
        contentDescription = "รูปสินค้า",
        placeholder = painterResource(R.drawable.ic_placeholder),
        error = painterResource(R.drawable.ic_error),
        fallback = painterResource(R.drawable.ic_fallback), // เมื่อ model = null
        modifier = Modifier
            .fillMaxWidth()
            .height(200.dp)
            .clip(RoundedCornerShape(12.dp)),
        contentScale = ContentScale.Crop
    )
}

// วิธีที่ 2: SubcomposeAsyncImage - control loading state แบบละเอียด
@Composable
fun SubcomposeAsyncImageExample() {
    SubcomposeAsyncImage(
        model = "https://picsum.photos/400/300",
        contentDescription = "รูป",
        modifier = Modifier
            .fillMaxWidth()
            .height(200.dp)
    ) {
        when (painter.state) {
            is AsyncImagePainter.State.Loading -> {
                // แสดง shimmer หรือ progress
                Box(
                    modifier = Modifier.fillMaxSize().background(Color.LightGray),
                    contentAlignment = Alignment.Center
                ) {
                    CircularProgressIndicator()
                }
            }
            is AsyncImagePainter.State.Error -> {
                Box(
                    modifier = Modifier.fillMaxSize().background(Color(0xFFF5F5F5)),
                    contentAlignment = Alignment.Center
                ) {
                    Column(horizontalAlignment = Alignment.CenterHorizontally) {
                        Icon(
                            Icons.Default.BrokenImage,
                            contentDescription = null,
                            tint = Color.Gray
                        )
                        Text("โหลดรูปไม่ได้", color = Color.Gray)
                    }
                }
            }
            else -> {
                SubcomposeAsyncImageContent(
                    modifier = Modifier.clip(RoundedCornerShape(12.dp)),
                    contentScale = ContentScale.Crop
                )
            }
        }
    }
}
```

---

## ขั้นตอนที่ 928: ImageRequest Builder

```kotlin
import coil.request.ImageRequest
import coil.size.Size
import coil.transform.CircleCropTransformation
import coil.transform.RoundedCornersTransformation
import coil.transform.BlurTransformation
import coil.transform.GrayscaleTransformation

@Composable
fun AdvancedCoilImages() {
    val context = LocalContext.current

    Column(verticalArrangement = Arrangement.spacedBy(16.dp)) {
        // รูปทรงกลม
        AsyncImage(
            model = ImageRequest.Builder(context)
                .data("https://i.pravatar.cc/200")
                .transformations(CircleCropTransformation())
                .build(),
            contentDescription = "Avatar",
            modifier = Modifier.size(80.dp)
        )

        // รูปมุมมน
        AsyncImage(
            model = ImageRequest.Builder(context)
                .data("https://picsum.photos/300/200")
                .transformations(RoundedCornersTransformation(radius = 24f))
                .build(),
            contentDescription = "Rounded",
            modifier = Modifier.fillMaxWidth().height(150.dp),
            contentScale = ContentScale.Crop
        )

        // รูป Blur
        AsyncImage(
            model = ImageRequest.Builder(context)
                .data("https://picsum.photos/300/200")
                .transformations(BlurTransformation(context, radius = 15f))
                .build(),
            contentDescription = "Blurred",
            modifier = Modifier.fillMaxWidth().height(150.dp),
            contentScale = ContentScale.Crop
        )

        // รูปขาวดำ
        AsyncImage(
            model = ImageRequest.Builder(context)
                .data("https://picsum.photos/300/200")
                .transformations(GrayscaleTransformation())
                .build(),
            contentDescription = "Grayscale",
            modifier = Modifier.fillMaxWidth().height(150.dp),
            contentScale = ContentScale.Crop
        )

        // รูปขนาดเล็ก (thumbnail) สำหรับ list
        AsyncImage(
            model = ImageRequest.Builder(context)
                .data("https://picsum.photos/300/200")
                .size(100, 100) // บอก Coil ว่าต้องการขนาดนี้ (ประหยัด memory)
                .memoryCacheKey("thumbnail_${1}")
                .build(),
            contentDescription = "Thumbnail",
            modifier = Modifier.size(60.dp).clip(CircleShape),
            contentScale = ContentScale.Crop
        )
    }
}
```

---

## ขั้นตอนที่ 929: ImageLoader Configuration

```kotlin
import coil.ImageLoader
import coil.disk.DiskCache
import coil.memory.MemoryCache
import coil.util.DebugLogger
import okhttp3.OkHttpClient

// ตั้งค่า ImageLoader สำหรับทั้ง App
@Composable
fun rememberCustomImageLoader(): ImageLoader {
    val context = LocalContext.current
    return remember {
        ImageLoader.Builder(context)
            // Memory Cache
            .memoryCache {
                MemoryCache.Builder(context)
                    .maxSizePercent(0.25) // ใช้ 25% ของ memory
                    .build()
            }
            // Disk Cache
            .diskCache {
                DiskCache.Builder()
                    .directory(context.cacheDir.resolve("image_cache"))
                    .maxSizePercent(0.02) // ใช้ 2% ของ disk
                    .build()
            }
            // OkHttpClient (ใช้ instance เดียวกับ Retrofit)
            .okHttpClient {
                OkHttpClient.Builder()
                    .addInterceptor { chain ->
                        val request = chain.request().newBuilder()
                            .addHeader("Authorization", "Bearer TOKEN")
                            .build()
                        chain.proceed(request)
                    }
                    .build()
            }
            // Logger
            .logger(DebugLogger())
            // GIF support
            .components {
                add(GifDecoder.Factory())
            }
            .crossfade(true)
            .build()
    }
}

// ใช้ใน MainActivity หรือ Composable
@Composable
fun ImageApp() {
    val imageLoader = rememberCustomImageLoader()

    // ส่ง imageLoader ไปยัง AsyncImage
    AsyncImage(
        model = "https://media.giphy.com/media/xxx/giphy.gif",
        contentDescription = "GIF Animation",
        imageLoader = imageLoader
    )
}

// Preload รูป
@Composable
fun PreloadImages(urls: List<String>) {
    val context = LocalContext.current
    LaunchedEffect(urls) {
        urls.forEach { url ->
            val request = ImageRequest.Builder(context)
                .data(url)
                .build()
            context.imageLoader.enqueue(request)
        }
    }
}
```

---

## ขั้นตอนที่ 930: Image Gallery ด้วย Coil

```kotlin
import coil.compose.AsyncImage
import androidx.compose.foundation.clickable
import androidx.compose.foundation.layout.*
import androidx.compose.foundation.lazy.grid.*
import androidx.compose.material3.*

@Composable
fun ImageGallery() {
    val imageUrls = (1..50).map { "https://picsum.photos/seed/$it/300/300" }
    var selectedImage by remember { mutableStateOf<String?>(null) }

    LazyVerticalGrid(
        columns = GridCells.Fixed(3),
        contentPadding = PaddingValues(2.dp),
        horizontalArrangement = Arrangement.spacedBy(2.dp),
        verticalArrangement = Arrangement.spacedBy(2.dp)
    ) {
        items(imageUrls, key = { it }) { url ->
            AsyncImage(
                model = ImageRequest.Builder(LocalContext.current)
                    .data(url)
                    .crossfade(true)
                    .build(),
                contentDescription = null,
                modifier = Modifier
                    .aspectRatio(1f) // square
                    .clickable { selectedImage = url },
                contentScale = ContentScale.Crop
            )
        }
    }

    // Full-screen image viewer
    selectedImage?.let { url ->
        Dialog(onDismissRequest = { selectedImage = null }) {
            Box(
                modifier = Modifier
                    .fillMaxSize()
                    .background(Color.Black)
                    .clickable { selectedImage = null }
            ) {
                AsyncImage(
                    model = url,
                    contentDescription = "รูปเต็มขนาด",
                    modifier = Modifier.fillMaxSize(),
                    contentScale = ContentScale.Fit
                )
            }
        }
    }
}

// Avatar พร้อม fallback ตัวอักษร
@Composable
fun UserAvatar(name: String, imageUrl: String?, size: Dp = 40.dp) {
    Box(
        modifier = Modifier
            .size(size)
            .clip(CircleShape)
            .background(MaterialTheme.colorScheme.primaryContainer),
        contentAlignment = Alignment.Center
    ) {
        if (imageUrl != null) {
            AsyncImage(
                model = imageUrl,
                contentDescription = name,
                modifier = Modifier.fillMaxSize(),
                contentScale = ContentScale.Crop
            )
        } else {
            Text(
                text = name.take(1).uppercase(),
                style = MaterialTheme.typography.titleMedium,
                color = MaterialTheme.colorScheme.onPrimaryContainer
            )
        }
    }
}
```

---

*Part 46 จบแล้ว | ก่อนหน้า: [Part 45](../part45/README.md) | ถัดไป: [Part 47](../part47/README.md)*
