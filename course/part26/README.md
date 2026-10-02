# Part 26: Retrofit และ Networking
## ขั้นตอนที่ 626-650

---

## ขั้นตอนที่ 626: Retrofit Setup

```kotlin
// build.gradle.kts
// implementation("com.squareup.retrofit2:retrofit:2.11.x")
// implementation("com.squareup.retrofit2:converter-gson:2.11.x")
// implementation("com.squareup.okhttp3:okhttp:4.12.x")
// implementation("com.squareup.okhttp3:logging-interceptor:4.12.x")

// libs.versions.toml
// [versions]
// retrofit = "2.11.0"
// okhttp = "4.12.0"
// [libraries]
// retrofit = { module = "com.squareup.retrofit2:retrofit", version.ref = "retrofit" }
// retrofit-gson = { module = "com.squareup.retrofit2:converter-gson", version.ref = "retrofit" }
// okhttp = { module = "com.squareup.okhttp3:okhttp", version.ref = "okhttp" }
// okhttp-logging = { module = "com.squareup.okhttp3:logging-interceptor", version.ref = "okhttp" }
```

---

## ขั้นตอนที่ 627: API Service Interface

```kotlin
// ============================================
// Data Models
// ============================================

data class Post(
    @SerializedName("id") val id: Long,
    @SerializedName("userId") val userId: Long,
    @SerializedName("title") val title: String,
    @SerializedName("body") val body: String
)

data class User(
    @SerializedName("id") val id: Long,
    @SerializedName("name") val name: String,
    @SerializedName("email") val email: String,
    @SerializedName("phone") val phone: String,
    @SerializedName("website") val website: String,
    @SerializedName("address") val address: Address,
    @SerializedName("company") val company: Company
)

data class Address(
    @SerializedName("street") val street: String,
    @SerializedName("suite") val suite: String,
    @SerializedName("city") val city: String,
    @SerializedName("zipcode") val zipcode: String
)

data class Company(
    @SerializedName("name") val name: String,
    @SerializedName("catchPhrase") val catchPhrase: String
)

// Request/Response wrappers
data class CreatePostRequest(
    @SerializedName("title") val title: String,
    @SerializedName("body") val body: String,
    @SerializedName("userId") val userId: Long
)

data class ApiResponse<T>(
    @SerializedName("data") val data: T?,
    @SerializedName("message") val message: String?,
    @SerializedName("success") val success: Boolean
)

data class PaginatedResponse<T>(
    @SerializedName("data") val data: List<T>,
    @SerializedName("total") val total: Int,
    @SerializedName("page") val page: Int,
    @SerializedName("limit") val limit: Int,
    @SerializedName("totalPages") val totalPages: Int
)

// ============================================
// API Service Interface
// ============================================

interface ApiService {
    // GET - List
    @GET("posts")
    suspend fun getPosts(): List<Post>
    
    @GET("posts")
    suspend fun getPostsPaged(
        @Query("page") page: Int,
        @Query("limit") limit: Int = 20
    ): PaginatedResponse<Post>
    
    @GET("posts")
    suspend fun getPostsByUser(@Query("userId") userId: Long): List<Post>
    
    // GET - Single
    @GET("posts/{id}")
    suspend fun getPost(@Path("id") id: Long): Post
    
    // GET - Search
    @GET("posts/search")
    suspend fun searchPosts(
        @Query("q") query: String,
        @Query("page") page: Int = 1,
        @Query("limit") limit: Int = 20
    ): PaginatedResponse<Post>
    
    // POST - Create
    @POST("posts")
    suspend fun createPost(@Body request: CreatePostRequest): Post
    
    // PUT - Full Update
    @PUT("posts/{id}")
    suspend fun updatePost(
        @Path("id") id: Long,
        @Body request: CreatePostRequest
    ): Post
    
    // PATCH - Partial Update
    @PATCH("posts/{id}")
    suspend fun patchPost(
        @Path("id") id: Long,
        @Body fields: Map<String, @JvmSuppressWildcards Any>
    ): Post
    
    // DELETE
    @DELETE("posts/{id}")
    suspend fun deletePost(@Path("id") id: Long): Response<Unit>
    
    // Headers
    @GET("secure/data")
    suspend fun getSecureData(
        @Header("Authorization") token: String
    ): ApiResponse<Any>
    
    // File Upload
    @Multipart
    @POST("upload")
    suspend fun uploadFile(
        @Part("description") description: RequestBody,
        @Part file: MultipartBody.Part
    ): ApiResponse<String>
    
    // Users
    @GET("users")
    suspend fun getUsers(): List<User>
    
    @GET("users/{id}")
    suspend fun getUser(@Path("id") id: Long): User
}
```

---

## ขั้นตอนที่ 628: Retrofit Client Setup

```kotlin
// ============================================
// OkHttp Interceptors
// ============================================

class AuthInterceptor(
    private val tokenProvider: () -> String?
) : Interceptor {
    override fun intercept(chain: Interceptor.Chain): Response {
        val original = chain.request()
        val token = tokenProvider()
        
        val request = if (token != null) {
            original.newBuilder()
                .header("Authorization", "Bearer $token")
                .build()
        } else {
            original
        }
        
        return chain.proceed(request)
    }
}

class RetryInterceptor(private val maxRetries: Int = 3) : Interceptor {
    override fun intercept(chain: Interceptor.Chain): Response {
        var currentTry = 0
        var response = chain.proceed(chain.request())
        
        while (!response.isSuccessful && currentTry < maxRetries) {
            if (response.code !in listOf(408, 500, 502, 503, 504)) break
            
            response.close()
            currentTry++
            
            Thread.sleep(1000L * currentTry)  // exponential backoff
            response = chain.proceed(chain.request())
        }
        
        return response
    }
}

class CacheInterceptor : Interceptor {
    override fun intercept(chain: Interceptor.Chain): Response {
        val request = chain.request()
        val response = chain.proceed(request)
        
        // cache for 5 minutes
        val cacheControl = CacheControl.Builder()
            .maxAge(5, TimeUnit.MINUTES)
            .build()
        
        return response.newBuilder()
            .header("Cache-Control", cacheControl.toString())
            .build()
    }
}

// ============================================
// Retrofit Instance
// ============================================

object RetrofitClient {
    
    private const val BASE_URL = "https://jsonplaceholder.typicode.com/"
    
    private fun createOkHttpClient(
        tokenProvider: () -> String? = { null }
    ): OkHttpClient {
        return OkHttpClient.Builder()
            .connectTimeout(30, TimeUnit.SECONDS)
            .readTimeout(30, TimeUnit.SECONDS)
            .writeTimeout(30, TimeUnit.SECONDS)
            .addInterceptor(AuthInterceptor(tokenProvider))
            .addInterceptor(RetryInterceptor(maxRetries = 3))
            .addInterceptor(CacheInterceptor())
            .addInterceptor(
                HttpLoggingInterceptor().apply {
                    level = if (BuildConfig.DEBUG) {
                        HttpLoggingInterceptor.Level.BODY
                    } else {
                        HttpLoggingInterceptor.Level.NONE
                    }
                }
            )
            .build()
    }
    
    fun createRetrofit(tokenProvider: () -> String? = { null }): Retrofit {
        return Retrofit.Builder()
            .baseUrl(BASE_URL)
            .client(createOkHttpClient(tokenProvider))
            .addConverterFactory(GsonConverterFactory.create(createGson()))
            .build()
    }
    
    private fun createGson(): Gson {
        return GsonBuilder()
            .setDateFormat("yyyy-MM-dd'T'HH:mm:ss.SSS'Z'")
            .setFieldNamingPolicy(FieldNamingPolicy.LOWER_CASE_WITH_UNDERSCORES)
            .create()
    }
    
    fun createApiService(tokenProvider: () -> String? = { null }): ApiService {
        return createRetrofit(tokenProvider).create(ApiService::class.java)
    }
}
```

---

## ขั้นตอนที่ 629: Error Handling

```kotlin
// ============================================
// Network Error Types
// ============================================

sealed class NetworkError : Exception() {
    data class HttpError(val code: Int, override val message: String?) : NetworkError()
    data class NetworkConnectionError(override val cause: Throwable?) : NetworkError()
    data class TimeoutError(override val cause: Throwable?) : NetworkError()
    data class ParseError(override val cause: Throwable?) : NetworkError()
    data class UnknownError(override val cause: Throwable?) : NetworkError()
}

// ============================================
// Safe API Call Helper
// ============================================

suspend fun <T> safeApiCall(call: suspend () -> T): Result<T> {
    return try {
        Result.Success(call())
    } catch (e: CancellationException) {
        throw e  // never catch CancellationException
    } catch (e: HttpException) {
        val errorBody = e.response()?.errorBody()?.string()
        Result.Error(NetworkError.HttpError(e.code(), errorBody ?: e.message()))
    } catch (e: SocketTimeoutException) {
        Result.Error(NetworkError.TimeoutError(e))
    } catch (e: IOException) {
        Result.Error(NetworkError.NetworkConnectionError(e))
    } catch (e: JsonSyntaxException) {
        Result.Error(NetworkError.ParseError(e))
    } catch (e: Exception) {
        Result.Error(NetworkError.UnknownError(e))
    }
}

// ============================================
// Repository ที่ใช้ Error Handling
// ============================================

class PostRepository(private val apiService: ApiService) {
    
    suspend fun getPosts(): Result<List<Post>> = safeApiCall {
        apiService.getPosts()
    }
    
    suspend fun getPost(id: Long): Result<Post> = safeApiCall {
        apiService.getPost(id)
    }
    
    suspend fun createPost(title: String, body: String, userId: Long): Result<Post> = safeApiCall {
        apiService.createPost(CreatePostRequest(title, body, userId))
    }
    
    // Flow-based dengan error handling
    fun observePosts(): Flow<Result<List<Post>>> = flow {
        emit(Result.Loading)
        val result = safeApiCall { apiService.getPosts() }
        emit(result)
    }
    
    // Paginated
    suspend fun getPostsPaged(page: Int, limit: Int = 20): Result<PaginatedResponse<Post>> = safeApiCall {
        apiService.getPostsPaged(page, limit)
    }
}

// ViewModel ใช้ Result
class PostsViewModel(private val repository: PostRepository) : ViewModel() {
    
    private val _uiState = MutableStateFlow<PostsUiState>(PostsUiState.Loading)
    val uiState: StateFlow<PostsUiState> = _uiState.asStateFlow()
    
    init { loadPosts() }
    
    fun loadPosts() {
        viewModelScope.launch {
            _uiState.value = PostsUiState.Loading
            
            when (val result = repository.getPosts()) {
                is Result.Success -> _uiState.value = PostsUiState.Success(result.data)
                is Result.Error -> _uiState.value = PostsUiState.Error(
                    when (val error = result.exception) {
                        is NetworkError.NetworkConnectionError -> "ไม่มีอินเทอร์เน็ต"
                        is NetworkError.TimeoutError -> "หมดเวลาการเชื่อมต่อ"
                        is NetworkError.HttpError -> "เกิดข้อผิดพลาด (${error.code})"
                        else -> "เกิดข้อผิดพลาด"
                    }
                )
                else -> {}
            }
        }
    }
}

sealed class PostsUiState {
    object Loading : PostsUiState()
    data class Success(val posts: List<Post>) : PostsUiState()
    data class Error(val message: String) : PostsUiState()
}
```

---

## ขั้นตอนที่ 630: Caching Strategy

```kotlin
// ============================================
// Repository ที่มี Cache (offline-first)
// ============================================

class OfflineFirstPostRepository(
    private val apiService: ApiService,
    private val postDao: PostDao,
    private val ioDispatcher: CoroutineDispatcher = Dispatchers.IO
) {
    
    // Observe from local + sync from network
    fun observePosts(): Flow<List<Post>> = flow {
        // 1. Emit local data immediately
        emitAll(postDao.observeAllPosts().map { entities ->
            entities.map { it.toPost() }
        })
    }.also {
        // 2. Sync from network in background
        viewModelScope.launch {
            syncPosts()
        }
    }
    
    suspend fun syncPosts() {
        withContext(ioDispatcher) {
            try {
                val remotePosts = apiService.getPosts()
                postDao.deleteAllPosts()
                postDao.insertPosts(remotePosts.map { it.toEntity() })
            } catch (e: Exception) {
                // Use cached data if network fails
            }
        }
    }
    
    // Unified flow: local + remote
    fun getPostsFlow(): Flow<Resource<List<Post>>> = flow {
        emit(Resource.Loading(data = null))
        
        // Emit cached
        val cached = postDao.getAllPosts().map { it.toPost() }
        emit(Resource.Loading(data = cached))
        
        // Fetch from network
        try {
            val response = withContext(ioDispatcher) { apiService.getPosts() }
            postDao.deleteAllPosts()
            postDao.insertPosts(response.map { it.toEntity() })
        } catch (e: Exception) {
            emit(Resource.Error(
                message = e.message ?: "Error",
                data = cached
            ))
        }
        
        // Emit final local data
        emitAll(postDao.observeAllPosts().map { entities ->
            Resource.Success(entities.map { it.toPost() })
        })
    }
}

// Resource wrapper with optional data
sealed class Resource<T>(
    val data: T? = null,
    val message: String? = null
) {
    class Success<T>(data: T) : Resource<T>(data)
    class Error<T>(message: String, data: T? = null) : Resource<T>(data, message)
    class Loading<T>(data: T? = null) : Resource<T>(data)
}
```

---

## ขั้นตอนที่ 631: Hilt DI สำหรับ Networking

```kotlin
@Module
@InstallIn(SingletonComponent::class)
object NetworkModule {
    
    @Provides
    @Singleton
    fun provideGson(): Gson = GsonBuilder()
        .setDateFormat("yyyy-MM-dd'T'HH:mm:ss.SSS'Z'")
        .create()
    
    @Provides
    @Singleton
    fun provideOkHttpClient(
        authInterceptor: AuthInterceptor
    ): OkHttpClient = OkHttpClient.Builder()
        .connectTimeout(30, TimeUnit.SECONDS)
        .readTimeout(30, TimeUnit.SECONDS)
        .addInterceptor(authInterceptor)
        .addInterceptor(HttpLoggingInterceptor().apply {
            level = if (BuildConfig.DEBUG)
                HttpLoggingInterceptor.Level.BODY
            else
                HttpLoggingInterceptor.Level.NONE
        })
        .build()
    
    @Provides
    @Singleton
    fun provideRetrofit(
        okHttpClient: OkHttpClient,
        gson: Gson
    ): Retrofit = Retrofit.Builder()
        .baseUrl(BuildConfig.BASE_URL)
        .client(okHttpClient)
        .addConverterFactory(GsonConverterFactory.create(gson))
        .build()
    
    @Provides
    @Singleton
    fun provideApiService(retrofit: Retrofit): ApiService =
        retrofit.create(ApiService::class.java)
    
    @Provides
    @Singleton
    fun provideAuthInterceptor(
        tokenDataStore: TokenDataStore
    ): AuthInterceptor = AuthInterceptor {
        runBlocking { tokenDataStore.getToken() }
    }
}
```

---

## แบบฝึกหัด Part 26

```kotlin
// แบบฝึกหัด: GitHub API Client
// ใช้ GitHub API เพื่อค้นหา Repository

data class GitHubRepo(
    @SerializedName("id") val id: Long,
    @SerializedName("name") val name: String,
    @SerializedName("full_name") val fullName: String,
    @SerializedName("description") val description: String?,
    @SerializedName("stargazers_count") val stars: Int,
    @SerializedName("language") val language: String?,
    @SerializedName("html_url") val url: String
)

data class GitHubSearchResponse(
    @SerializedName("total_count") val totalCount: Int,
    @SerializedName("items") val items: List<GitHubRepo>
)

interface GitHubApiService {
    @GET("search/repositories")
    suspend fun searchRepos(
        @Query("q") query: String,
        @Query("sort") sort: String = "stars",
        @Query("per_page") perPage: Int = 20,
        @Query("page") page: Int = 1
    ): GitHubSearchResponse
    
    @GET("repos/{owner}/{repo}")
    suspend fun getRepo(
        @Path("owner") owner: String,
        @Path("repo") repo: String
    ): GitHubRepo
}

// TODO: สร้าง GitHubRepository ที่มี:
// - searchRepos(query, page)
// - getRepo(owner, repo)
// - Error handling ที่ครบถ้วน
// - Caching ด้วย Room

// TODO: สร้าง ViewModel และ UI สำหรับ GitHub Search
```

---

*Part 26 จบแล้ว | ก่อนหน้า: [Part 25](../part25/README.md) | ถัดไป: [Part 27](../part27/README.md)*
