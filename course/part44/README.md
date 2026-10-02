# Part 44: Android - Repository Pattern (Room + Retrofit)
## ขั้นตอนที่ 876-900

---

## ขั้นตอนที่ 876: Offline-First Architecture

Offline-First หมายถึง app ทำงานได้แม้ไม่มีอินเทอร์เน็ต โดยดึงจาก local DB ก่อน แล้ว sync กับ API

```kotlin
// Single Source of Truth: Room เป็น source เดียว
// Flow จาก Room → UI อัพเดทอัตโนมัติเมื่อ database เปลี่ยน
// API call → update Room → UI อัพเดท

// Entity (Room)
@Entity(tableName = "articles")
data class ArticleEntity(
    @PrimaryKey val id: Int,
    val title: String,
    val content: String,
    val author: String,
    val publishedAt: String,
    val imageUrl: String,
    val isFavorite: Boolean = false,
    val lastFetched: Long = System.currentTimeMillis()
)

// API Response Model
data class ArticleDto(
    val id: Int,
    val title: String,
    val content: String,
    val author: String,
    @SerializedName("published_at") val publishedAt: String,
    @SerializedName("image_url") val imageUrl: String
)

// Mapper - แปลง DTO ↔ Entity ↔ Domain
fun ArticleDto.toEntity(): ArticleEntity = ArticleEntity(
    id = id, title = title, content = content,
    author = author, publishedAt = publishedAt, imageUrl = imageUrl
)

// Domain Model (อิสระจาก Room และ Retrofit)
data class Article(
    val id: Int, val title: String, val content: String,
    val author: String, val publishedAt: String,
    val imageUrl: String, val isFavorite: Boolean
)

fun ArticleEntity.toDomain(): Article = Article(
    id = id, title = title, content = content,
    author = author, publishedAt = publishedAt,
    imageUrl = imageUrl, isFavorite = isFavorite
)
```

---

## ขั้นตอนที่ 877: Dao และ API Interface

```kotlin
// Dao
@Dao
interface ArticleDao {
    @Query("SELECT * FROM articles ORDER BY published_at DESC")
    fun getAllArticles(): Flow<List<ArticleEntity>>

    @Query("SELECT * FROM articles WHERE is_favorite = 1")
    fun getFavoriteArticles(): Flow<List<ArticleEntity>>

    @Query("SELECT * FROM articles WHERE id = :id")
    fun getArticleById(id: Int): Flow<ArticleEntity?>

    @Insert(onConflict = OnConflictStrategy.REPLACE)
    suspend fun insertArticles(articles: List<ArticleEntity>)

    @Query("UPDATE articles SET is_favorite = :isFavorite WHERE id = :id")
    suspend fun updateFavorite(id: Int, isFavorite: Boolean)

    @Query("DELETE FROM articles")
    suspend fun deleteAll()
}

// API Interface
interface NewsApi {
    @GET("articles")
    suspend fun getArticles(
        @Query("page") page: Int = 1,
        @Query("per_page") perPage: Int = 20
    ): NewsResponse

    @GET("articles/{id}")
    suspend fun getArticleById(@Path("id") id: Int): ArticleDto
}

data class NewsResponse(
    val articles: List<ArticleDto>,
    val total: Int,
    val page: Int,
    val totalPages: Int
)
```

---

## ขั้นตอนที่ 878: Repository Implementation แบบ Offline-First

```kotlin
interface ArticleRepository {
    fun getArticles(): Flow<Resource<List<Article>>>
    fun getFavorites(): Flow<List<Article>>
    suspend fun refreshArticles()
    suspend fun toggleFavorite(articleId: Int)
}

// Resource wrapper
sealed class Resource<T> {
    data class Success<T>(val data: T) : Resource<T>()
    data class Loading<T>(val data: T? = null) : Resource<T>() // data = cached data
    data class Error<T>(val message: String, val data: T? = null) : Resource<T>()
}

class ArticleRepositoryImpl(
    private val articleDao: ArticleDao,
    private val newsApi: NewsApi
) : ArticleRepository {

    // networkBoundResource - pattern สำหรับ offline-first
    override fun getArticles(): Flow<Resource<List<Article>>> = flow {
        // 1. emit Loading พร้อม cached data
        val cachedArticles = articleDao.getAllArticles().first()
        emit(Resource.Loading(cachedArticles.map { it.toDomain() }))

        // 2. ดึงจาก network
        try {
            val response = newsApi.getArticles()
            val entities = response.articles.map { it.toEntity() }
            articleDao.insertArticles(entities)
        } catch (e: Exception) {
            // network ล้มเหลว - emit error พร้อม cached data
            emit(Resource.Error(
                message = e.message ?: "ไม่สามารถโหลดข้อมูลได้",
                data = articleDao.getAllArticles().first().map { it.toDomain() }
            ))
        }

        // 3. emit data จาก Room (ซึ่งอัพเดทแล้ว)
        emitAll(
            articleDao.getAllArticles().map { entities ->
                Resource.Success(entities.map { it.toDomain() })
            }
        )
    }

    override fun getFavorites(): Flow<List<Article>> =
        articleDao.getFavoriteArticles().map { it.map { e -> e.toDomain() } }

    override suspend fun refreshArticles() {
        val response = newsApi.getArticles()
        articleDao.insertArticles(response.articles.map { it.toEntity() })
    }

    override suspend fun toggleFavorite(articleId: Int) {
        val current = articleDao.getArticleById(articleId).first()
        current?.let {
            articleDao.updateFavorite(articleId, !it.isFavorite)
        }
    }
}
```

---

## ขั้นตอนที่ 879: ViewModel กับ Repository

```kotlin
class ArticleViewModel(
    private val repository: ArticleRepository
) : ViewModel() {

    val articles: StateFlow<Resource<List<Article>>> = repository
        .getArticles()
        .stateIn(
            scope = viewModelScope,
            started = SharingStarted.WhileSubscribed(5000),
            initialValue = Resource.Loading()
        )

    val favorites: StateFlow<List<Article>> = repository
        .getFavorites()
        .stateIn(viewModelScope, SharingStarted.WhileSubscribed(5000), emptyList())

    fun refresh() {
        viewModelScope.launch {
            try {
                repository.refreshArticles()
            } catch (e: Exception) {
                // handle error
            }
        }
    }

    fun toggleFavorite(articleId: Int) {
        viewModelScope.launch {
            repository.toggleFavorite(articleId)
        }
    }
}

@OptIn(ExperimentalMaterial3Api::class)
@Composable
fun ArticleListScreen(viewModel: ArticleViewModel = hiltViewModel()) {
    val articlesState by viewModel.articles.collectAsStateWithLifecycle()
    val pullRefreshState = rememberPullToRefreshState()

    if (pullRefreshState.isRefreshing) {
        LaunchedEffect(true) {
            viewModel.refresh()
            pullRefreshState.endRefresh()
        }
    }

    Box(Modifier.fillMaxSize().nestedScroll(pullRefreshState.nestedScrollConnection)) {
        when (val state = articlesState) {
            is Resource.Loading -> {
                if (state.data != null) {
                    // แสดง cached data ขณะโหลด
                    ArticleList(articles = state.data!!, viewModel = viewModel)
                } else {
                    Box(Modifier.fillMaxSize(), Alignment.Center) {
                        CircularProgressIndicator()
                    }
                }
            }
            is Resource.Success -> {
                ArticleList(articles = state.data, viewModel = viewModel)
            }
            is Resource.Error -> {
                Column(Modifier.fillMaxSize(), Arrangement.Center, Alignment.CenterHorizontally) {
                    Text(state.message, color = MaterialTheme.colorScheme.error)
                    state.data?.let { cached ->
                        Text("แสดงข้อมูลเก่า:")
                        ArticleList(articles = cached, viewModel = viewModel)
                    }
                    Button(onClick = { viewModel.refresh() }) { Text("ลองใหม่") }
                }
            }
        }
        PullToRefreshContainer(state = pullRefreshState, modifier = Modifier.align(Alignment.TopCenter))
    }
}

@Composable
fun ArticleList(articles: List<Article>, viewModel: ArticleViewModel) {
    LazyColumn {
        items(articles, key = { it.id }) { article ->
            ArticleCard(
                article = article,
                onToggleFavorite = { viewModel.toggleFavorite(article.id) }
            )
        }
    }
}
```

---

## ขั้นตอนที่ 880: Hilt Setup สำหรับ Repository Pattern

```kotlin
// AppModule.kt
@Module
@InstallIn(SingletonComponent::class)
object AppModule {
    @Provides
    @Singleton
    fun provideDatabase(@ApplicationContext context: Context): AppDatabase =
        Room.databaseBuilder(context, AppDatabase::class.java, "news_db").build()

    @Provides
    fun provideArticleDao(db: AppDatabase): ArticleDao = db.articleDao()
}

@Module
@InstallIn(SingletonComponent::class)
object NetworkModule {
    @Provides
    @Singleton
    fun provideNewsApi(): NewsApi {
        return Retrofit.Builder()
            .baseUrl("https://newsapi.example.com/")
            .addConverterFactory(GsonConverterFactory.create())
            .build()
            .create(NewsApi::class.java)
    }
}

@Module
@InstallIn(SingletonComponent::class)
abstract class RepositoryModule {
    @Binds
    @Singleton
    abstract fun bindArticleRepository(
        impl: ArticleRepositoryImpl
    ): ArticleRepository
}

// Application class
@HiltAndroidApp
class MyApplication : Application()

// MainActivity
@AndroidEntryPoint
class MainActivity : ComponentActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContent {
            AppTheme {
                // ArticleListScreen ใช้ hiltViewModel() ได้เลย
                ArticleListScreen()
            }
        }
    }
}
```

---

*Part 44 จบแล้ว | ก่อนหน้า: [Part 43](../part43/README.md) | ถัดไป: [Part 45](../part45/README.md)*
