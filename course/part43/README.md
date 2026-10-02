# Part 43: Android - Retrofit พื้นฐาน (Network Calls)
## ขั้นตอนที่ 851-875

---

## ขั้นตอนที่ 851: ติดตั้งและตั้งค่า Retrofit

Retrofit คือ HTTP client ยอดนิยมสำหรับ Android ช่วยเรียก REST API ได้ง่ายมาก

```kotlin
// build.gradle.kts (app)
dependencies {
    implementation("com.squareup.retrofit2:retrofit:2.9.0")
    implementation("com.squareup.retrofit2:converter-gson:2.9.0")
    implementation("com.squareup.okhttp3:okhttp:4.12.0")
    implementation("com.squareup.okhttp3:logging-interceptor:4.12.0")
    implementation("com.google.code.gson:gson:2.10.1")
}

// AndroidManifest.xml - ต้องขอ permission
// <uses-permission android:name="android.permission.INTERNET" />

// Data Models (จาก JSON API)
data class Post(
    val id: Int,
    val userId: Int,
    val title: String,
    val body: String
)

data class Comment(
    val id: Int,
    val postId: Int,
    val name: String,
    val email: String,
    val body: String
)

data class User(
    val id: Int,
    val name: String,
    val username: String,
    val email: String,
    val address: Address,
    val phone: String,
    val website: String
)

data class Address(
    val street: String,
    val suite: String,
    val city: String,
    val zipcode: String
)
```

---

## ขั้นตอนที่ 852: สร้าง API Interface

```kotlin
import retrofit2.Response
import retrofit2.http.*

// API Interface - ประกาศ endpoints ทั้งหมด
interface JsonPlaceholderApi {
    // GET ทั้งหมด
    @GET("posts")
    suspend fun getPosts(): List<Post>

    // GET ด้วย path parameter
    @GET("posts/{id}")
    suspend fun getPostById(@Path("id") id: Int): Post

    // GET ด้วย query parameter
    @GET("posts")
    suspend fun getPostsByUser(
        @Query("userId") userId: Int,
        @Query("_limit") limit: Int = 10,
        @Query("_page") page: Int = 1
    ): List<Post>

    // GET Comments ของ Post
    @GET("posts/{postId}/comments")
    suspend fun getComments(@Path("postId") postId: Int): List<Comment>

    // POST - สร้างรายการใหม่
    @POST("posts")
    suspend fun createPost(@Body post: CreatePostRequest): Post

    // PUT - แทนที่ทั้งหมด
    @PUT("posts/{id}")
    suspend fun updatePost(
        @Path("id") id: Int,
        @Body post: UpdatePostRequest
    ): Post

    // PATCH - อัพเดทบางส่วน
    @PATCH("posts/{id}")
    suspend fun patchPost(
        @Path("id") id: Int,
        @Body updates: Map<String, String>
    ): Post

    // DELETE
    @DELETE("posts/{id}")
    suspend fun deletePost(@Path("id") id: Int): Response<Unit>

    // GET พร้อม Header
    @GET("users")
    suspend fun getUsers(
        @Header("Authorization") token: String
    ): List<User>
}

data class CreatePostRequest(
    val title: String,
    val body: String,
    val userId: Int
)

data class UpdatePostRequest(
    val id: Int,
    val title: String,
    val body: String,
    val userId: Int
)
```

---

## ขั้นตอนที่ 853: สร้าง Retrofit Instance

```kotlin
import okhttp3.OkHttpClient
import okhttp3.logging.HttpLoggingInterceptor
import retrofit2.Retrofit
import retrofit2.converter.gson.GsonConverterFactory
import java.util.concurrent.TimeUnit

object RetrofitClient {
    private const val BASE_URL = "https://jsonplaceholder.typicode.com/"

    // Logging Interceptor - แสดง HTTP request/response ใน Logcat
    private val loggingInterceptor = HttpLoggingInterceptor().apply {
        level = if (BuildConfig.DEBUG)
            HttpLoggingInterceptor.Level.BODY
        else
            HttpLoggingInterceptor.Level.NONE
    }

    // Auth Interceptor - ใส่ token อัตโนมัติทุก request
    private val authInterceptor = okhttp3.Interceptor { chain ->
        val request = chain.request().newBuilder()
            .addHeader("Authorization", "Bearer YOUR_TOKEN_HERE")
            .addHeader("Accept", "application/json")
            .build()
        chain.proceed(request)
    }

    private val okHttpClient = OkHttpClient.Builder()
        .addInterceptor(loggingInterceptor)
        .addInterceptor(authInterceptor)
        .connectTimeout(30, TimeUnit.SECONDS)
        .readTimeout(30, TimeUnit.SECONDS)
        .writeTimeout(30, TimeUnit.SECONDS)
        .build()

    private val retrofit = Retrofit.Builder()
        .baseUrl(BASE_URL)
        .client(okHttpClient)
        .addConverterFactory(GsonConverterFactory.create())
        .build()

    val api: JsonPlaceholderApi = retrofit.create(JsonPlaceholderApi::class.java)
}

// ดีกว่า: ใช้ Hilt สำหรับ DI
@Module
@InstallIn(SingletonComponent::class)
object NetworkModule {
    @Provides
    @Singleton
    fun provideOkHttpClient(): OkHttpClient = OkHttpClient.Builder()
        .addInterceptor(HttpLoggingInterceptor().apply {
            level = HttpLoggingInterceptor.Level.BODY
        })
        .build()

    @Provides
    @Singleton
    fun provideRetrofit(okHttpClient: OkHttpClient): Retrofit = Retrofit.Builder()
        .baseUrl("https://jsonplaceholder.typicode.com/")
        .client(okHttpClient)
        .addConverterFactory(GsonConverterFactory.create())
        .build()

    @Provides
    @Singleton
    fun provideApi(retrofit: Retrofit): JsonPlaceholderApi =
        retrofit.create(JsonPlaceholderApi::class.java)
}
```

---

## ขั้นตอนที่ 854: จัดการ Error และ Result

```kotlin
// Wrapper สำหรับ API response
sealed class ApiResult<out T> {
    data class Success<T>(val data: T) : ApiResult<T>()
    data class Error(
        val code: Int? = null,
        val message: String
    ) : ApiResult<Nothing>()
    object Loading : ApiResult<Nothing>()
}

// Extension function สำหรับ safe API call
suspend fun <T> safeApiCall(
    call: suspend () -> T
): ApiResult<T> {
    return try {
        ApiResult.Success(call())
    } catch (e: HttpException) {
        ApiResult.Error(
            code = e.code(),
            message = when (e.code()) {
                400 -> "คำขอไม่ถูกต้อง"
                401 -> "ไม่ได้รับอนุญาต"
                403 -> "ถูกปฏิเสธการเข้าถึง"
                404 -> "ไม่พบข้อมูล"
                500 -> "เซิร์ฟเวอร์มีปัญหา"
                else -> e.message ?: "เกิดข้อผิดพลาด"
            }
        )
    } catch (e: IOException) {
        ApiResult.Error(message = "ไม่มีการเชื่อมต่ออินเทอร์เน็ต")
    } catch (e: Exception) {
        ApiResult.Error(message = e.message ?: "เกิดข้อผิดพลาดที่ไม่คาดคิด")
    }
}

// Repository ที่ใช้ safeApiCall
class PostRepository(private val api: JsonPlaceholderApi) {
    suspend fun getPosts(): ApiResult<List<Post>> = safeApiCall { api.getPosts() }
    suspend fun getPostById(id: Int): ApiResult<Post> = safeApiCall { api.getPostById(id) }
    suspend fun createPost(title: String, body: String, userId: Int): ApiResult<Post> =
        safeApiCall { api.createPost(CreatePostRequest(title, body, userId)) }
}
```

---

## ขั้นตอนที่ 855: ViewModel กับ Retrofit

```kotlin
class PostViewModel(private val repository: PostRepository) : ViewModel() {
    private val _postsState = MutableStateFlow<ApiResult<List<Post>>>(ApiResult.Loading)
    val postsState: StateFlow<ApiResult<List<Post>>> = _postsState.asStateFlow()

    init {
        loadPosts()
    }

    fun loadPosts() {
        viewModelScope.launch {
            _postsState.value = ApiResult.Loading
            _postsState.value = repository.getPosts()
        }
    }

    fun createPost(title: String, body: String) {
        viewModelScope.launch {
            when (val result = repository.createPost(title, body, userId = 1)) {
                is ApiResult.Success -> {
                    // รีโหลด list หลังสร้างสำเร็จ
                    loadPosts()
                }
                is ApiResult.Error -> {
                    // แสดง error message
                }
                else -> {}
            }
        }
    }
}

@Composable
fun PostListScreen(viewModel: PostViewModel = viewModel()) {
    val state by viewModel.postsState.collectAsStateWithLifecycle()

    when (val s = state) {
        is ApiResult.Loading -> {
            Box(Modifier.fillMaxSize(), contentAlignment = Alignment.Center) {
                CircularProgressIndicator()
            }
        }
        is ApiResult.Error -> {
            Column(
                Modifier.fillMaxSize(),
                horizontalAlignment = Alignment.CenterHorizontally,
                verticalArrangement = Arrangement.Center
            ) {
                Text(s.message, color = MaterialTheme.colorScheme.error)
                Button(onClick = { viewModel.loadPosts() }) {
                    Text("ลองใหม่")
                }
            }
        }
        is ApiResult.Success -> {
            LazyColumn {
                items(s.data, key = { it.id }) { post ->
                    Card(modifier = Modifier.fillMaxWidth().padding(8.dp)) {
                        Column(modifier = Modifier.padding(16.dp)) {
                            Text(post.title, style = MaterialTheme.typography.titleMedium)
                            Spacer(Modifier.height(4.dp))
                            Text(
                                post.body,
                                style = MaterialTheme.typography.bodySmall,
                                maxLines = 3,
                                overflow = TextOverflow.Ellipsis
                            )
                        }
                    }
                }
            }
        }
    }
}
```

---

*Part 43 จบแล้ว | ก่อนหน้า: [Part 42](../part42/README.md) | ถัดไป: [Part 44](../part44/README.md)*
