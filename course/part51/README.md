# Part 51: Clean Architecture
## ขั้นตอนที่ 751-775

---

## ขั้นตอนที่ 751: Clean Architecture คืออะไร?

Clean Architecture แยก code ออกเป็น layers ที่มี dependency ไหลทิศเดียว

```
Clean Architecture Layers:

  ┌─────────────────────────────────────────────┐
  │              Presentation Layer              │
  │  (UI, ViewModel, State, Navigation)          │
  └──────────────────┬──────────────────────────┘
                     │ depends on
  ┌──────────────────▼──────────────────────────┐
  │               Domain Layer                   │
  │  (Use Cases, Domain Models, Interfaces)      │
  │  - Pure Kotlin, no Android dependencies      │
  └──────────────────┬──────────────────────────┘
                     │ depends on
  ┌──────────────────▼──────────────────────────┐
  │                Data Layer                    │
  │  (Repositories, Data Sources, DTOs)          │
  │  (Network, Database, Cache)                  │
  └─────────────────────────────────────────────┘

Dependency Rule: Outer → Inner (never reversed)
Domain layer ไม่รู้จัก Data และ Presentation
```

---

## ขั้นตอนที่ 752: Domain Layer

```kotlin
// ============================================
// Domain Models - Pure Kotlin, no Android
// ============================================

// Domain model แยกจาก Entity และ DTO
data class Article(
    val id: String,
    val title: String,
    val content: String,
    val author: Author,
    val tags: List<String>,
    val publishedAt: LocalDateTime,
    val status: ArticleStatus,
    val viewCount: Long
)

data class Author(
    val id: String,
    val name: String,
    val bio: String,
    val avatarUrl: String?
)

enum class ArticleStatus { DRAFT, PUBLISHED, ARCHIVED }

// Value Objects
@JvmInline
value class ArticleId(val value: String) {
    init { require(value.isNotBlank()) { "Article ID cannot be blank" } }
}

@JvmInline
value class Email(val value: String) {
    init { require(isValid(value)) { "Invalid email: $value" } }
    companion object {
        fun isValid(email: String) = email.matches(
            Regex("^[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\\.[A-Za-z]{2,}$")
        )
    }
}

// ============================================
// Repository Interfaces (Domain defines contracts)
// ============================================

interface ArticleRepository {
    suspend fun getArticle(id: ArticleId): Result<Article>
    suspend fun getArticles(filter: ArticleFilter): Result<List<Article>>
    fun observeArticles(filter: ArticleFilter): Flow<List<Article>>
    suspend fun saveArticle(article: Article): Result<Article>
    suspend fun deleteArticle(id: ArticleId): Result<Unit>
}

data class ArticleFilter(
    val status: ArticleStatus? = null,
    val authorId: String? = null,
    val tags: List<String> = emptyList(),
    val searchQuery: String = "",
    val page: Int = 0,
    val pageSize: Int = 20
)
```

---

## ขั้นตอนที่ 753: Use Cases

```kotlin
// ============================================
// Use Cases = Business Logic
// ============================================

// Base Use Case
abstract class UseCase<in P, out R> {
    operator fun invoke(params: P): Flow<Result<R>> = flow {
        try {
            emit(Result.Loading)
            emit(execute(params))
        } catch (e: CancellationException) {
            throw e
        } catch (e: Exception) {
            emit(Result.Error(e))
        }
    }.flowOn(Dispatchers.IO)
    
    protected abstract suspend fun execute(params: P): Result<R>
}

// Suspend Use Case
abstract class SuspendUseCase<in P, out R> {
    suspend operator fun invoke(params: P): Result<R> {
        return try {
            withContext(Dispatchers.IO) { execute(params) }
        } catch (e: CancellationException) {
            throw e
        } catch (e: Exception) {
            Result.Error(e)
        }
    }
    
    protected abstract suspend fun execute(params: P): Result<R>
}

// ============================================
// Concrete Use Cases
// ============================================

class GetArticlesUseCase @Inject constructor(
    private val articleRepository: ArticleRepository
) {
    operator fun invoke(filter: ArticleFilter): Flow<Result<List<Article>>> = flow {
        emit(Result.Loading)
        try {
            articleRepository.observeArticles(filter).collect { articles ->
                // Apply business rules
                val filtered = articles
                    .filter { it.status == ArticleStatus.PUBLISHED || filter.status != null }
                    .sortedByDescending { it.publishedAt }
                
                emit(Result.Success(filtered))
            }
        } catch (e: Exception) {
            emit(Result.Error(e))
        }
    }
}

class PublishArticleUseCase @Inject constructor(
    private val articleRepository: ArticleRepository,
    private val notificationService: NotificationService
) : SuspendUseCase<PublishArticleUseCase.Params, Article>() {
    
    data class Params(
        val articleId: ArticleId,
        val notifySubscribers: Boolean = true
    )
    
    override suspend fun execute(params: Params): Result<Article> {
        // 1. Get article
        val article = articleRepository.getArticle(params.articleId)
            .getOrThrow()
        
        // 2. Validate business rules
        if (article.status == ArticleStatus.ARCHIVED) {
            return Result.Error(IllegalStateException("Cannot publish archived article"))
        }
        
        if (article.title.isBlank() || article.content.isBlank()) {
            return Result.Error(IllegalStateException("Article must have title and content"))
        }
        
        // 3. Update status
        val published = article.copy(
            status = ArticleStatus.PUBLISHED,
            publishedAt = LocalDateTime.now()
        )
        
        // 4. Save
        val saved = articleRepository.saveArticle(published).getOrThrow()
        
        // 5. Notify (side effect)
        if (params.notifySubscribers) {
            notificationService.notifyArticlePublished(saved)
        }
        
        return Result.Success(saved)
    }
}

class SearchArticlesUseCase @Inject constructor(
    private val articleRepository: ArticleRepository
) {
    operator fun invoke(query: String): Flow<Result<List<Article>>> {
        return articleRepository
            .observeArticles(ArticleFilter(searchQuery = query, status = ArticleStatus.PUBLISHED))
            .map { articles ->
                // Score and rank results
                val scored = articles.map { article ->
                    val titleScore = if (article.title.contains(query, ignoreCase = true)) 10 else 0
                    val tagScore = article.tags.count { it.contains(query, ignoreCase = true) } * 5
                    val contentScore = if (article.content.contains(query, ignoreCase = true)) 1 else 0
                    Pair(article, titleScore + tagScore + contentScore)
                }
                
                Result.Success(
                    scored
                        .filter { (_, score) -> score > 0 }
                        .sortedByDescending { (_, score) -> score }
                        .map { (article, _) -> article }
                )
            }
            .catch { e -> emit(Result.Error(e)) }
    }
}
```

---

## ขั้นตอนที่ 754: Data Layer

```kotlin
// ============================================
// Data Transfer Objects (DTO)
// ============================================

// Network DTO
data class ArticleDto(
    @SerializedName("id") val id: String,
    @SerializedName("title") val title: String,
    @SerializedName("content") val content: String,
    @SerializedName("author") val author: AuthorDto,
    @SerializedName("tags") val tags: List<String>,
    @SerializedName("published_at") val publishedAt: String?,
    @SerializedName("status") val status: String,
    @SerializedName("view_count") val viewCount: Long
)

data class AuthorDto(
    @SerializedName("id") val id: String,
    @SerializedName("name") val name: String,
    @SerializedName("bio") val bio: String,
    @SerializedName("avatar_url") val avatarUrl: String?
)

// Mappers
fun ArticleDto.toDomain(): Article = Article(
    id = id,
    title = title,
    content = content,
    author = author.toDomain(),
    tags = tags,
    publishedAt = publishedAt?.let { LocalDateTime.parse(it) } ?: LocalDateTime.now(),
    status = when (status) {
        "published" -> ArticleStatus.PUBLISHED
        "archived" -> ArticleStatus.ARCHIVED
        else -> ArticleStatus.DRAFT
    },
    viewCount = viewCount
)

fun Article.toDto(): ArticleDto = ArticleDto(
    id = id,
    title = title,
    content = content,
    author = author.toDto(),
    tags = tags,
    publishedAt = publishedAt.toString(),
    status = status.name.lowercase(),
    viewCount = viewCount
)

// ============================================
// Repository Implementation
// ============================================

class ArticleRepositoryImpl @Inject constructor(
    private val remoteDataSource: ArticleRemoteDataSource,
    private val localDataSource: ArticleLocalDataSource,
    @IoDispatcher private val ioDispatcher: CoroutineDispatcher
) : ArticleRepository {
    
    override suspend fun getArticle(id: ArticleId): Result<Article> = withContext(ioDispatcher) {
        safeApiCall {
            // Try cache first
            localDataSource.getArticle(id.value)?.toDomain()
                ?: run {
                    val article = remoteDataSource.getArticle(id.value).toDomain()
                    localDataSource.saveArticle(article.toEntity())
                    article
                }
        }
    }
    
    override fun observeArticles(filter: ArticleFilter): Flow<List<Article>> {
        return localDataSource
            .observeArticles(filter.toLocalFilter())
            .map { entities -> entities.map { it.toDomain() } }
            .onStart {
                // Refresh from network
                try {
                    val remote = remoteDataSource.getArticles(filter.toRemoteParams())
                    localDataSource.saveArticles(remote.map { it.toEntity() })
                } catch (e: Exception) {
                    // Use cache if network fails
                }
            }
    }
    
    override suspend fun saveArticle(article: Article): Result<Article> = withContext(ioDispatcher) {
        safeApiCall {
            val dto = remoteDataSource.saveArticle(article.toDto())
            val saved = dto.toDomain()
            localDataSource.saveArticle(saved.toEntity())
            saved
        }
    }
    
    override suspend fun deleteArticle(id: ArticleId): Result<Unit> = withContext(ioDispatcher) {
        safeApiCall {
            remoteDataSource.deleteArticle(id.value)
            localDataSource.deleteArticle(id.value)
        }
    }
    
    override suspend fun getArticles(filter: ArticleFilter): Result<List<Article>> = withContext(ioDispatcher) {
        safeApiCall {
            remoteDataSource.getArticles(filter.toRemoteParams()).map { it.toDomain() }
        }
    }
}
```

---

## ขั้นตอนที่ 755: Presentation Layer

```kotlin
// ============================================
// ViewModel ใช้ Use Cases
// ============================================

@HiltViewModel
class ArticleListViewModel @Inject constructor(
    private val getArticlesUseCase: GetArticlesUseCase,
    private val searchArticlesUseCase: SearchArticlesUseCase
) : ViewModel() {
    
    private val _filter = MutableStateFlow(ArticleFilter())
    
    private val _uiState = MutableStateFlow(ArticleListUiState())
    val uiState: StateFlow<ArticleListUiState> = _uiState.asStateFlow()
    
    private val _events = Channel<ArticleListEvent>()
    val events = _events.receiveAsFlow()
    
    init {
        observeArticles()
    }
    
    private fun observeArticles() {
        viewModelScope.launch {
            _filter.flatMapLatest { filter ->
                if (filter.searchQuery.isBlank()) {
                    getArticlesUseCase(filter)
                } else {
                    searchArticlesUseCase(filter.searchQuery)
                }
            }.collect { result ->
                when (result) {
                    is Result.Loading -> _uiState.update { it.copy(isLoading = true) }
                    is Result.Success -> _uiState.update { it.copy(
                        articles = result.data,
                        isLoading = false,
                        error = null
                    ) }
                    is Result.Error -> _uiState.update { it.copy(
                        isLoading = false,
                        error = result.exception.message
                    ) }
                }
            }
        }
    }
    
    fun onSearchQueryChanged(query: String) {
        _filter.update { it.copy(searchQuery = query) }
    }
    
    fun onFilterChanged(status: ArticleStatus?) {
        _filter.update { it.copy(status = status, page = 0) }
    }
    
    fun onArticleClicked(article: Article) {
        viewModelScope.launch {
            _events.send(ArticleListEvent.NavigateToDetail(article.id))
        }
    }
}

data class ArticleListUiState(
    val articles: List<Article> = emptyList(),
    val isLoading: Boolean = false,
    val error: String? = null
)

sealed class ArticleListEvent {
    data class NavigateToDetail(val articleId: String) : ArticleListEvent()
}
```

---

## ขั้นตอนที่ 756: Project Structure สำหรับ Clean Architecture

```
app/
├── src/main/java/com/example/
│   ├── data/
│   │   ├── local/
│   │   │   ├── dao/
│   │   │   │   └── ArticleDao.kt
│   │   │   ├── entity/
│   │   │   │   └── ArticleEntity.kt
│   │   │   └── AppDatabase.kt
│   │   ├── remote/
│   │   │   ├── api/
│   │   │   │   └── ArticleApiService.kt
│   │   │   └── dto/
│   │   │       └── ArticleDto.kt
│   │   ├── repository/
│   │   │   └── ArticleRepositoryImpl.kt
│   │   └── mapper/
│   │       └── ArticleMapper.kt
│   │
│   ├── domain/
│   │   ├── model/
│   │   │   ├── Article.kt
│   │   │   └── Author.kt
│   │   ├── repository/
│   │   │   └── ArticleRepository.kt      ← Interface
│   │   └── usecase/
│   │       ├── GetArticlesUseCase.kt
│   │       ├── PublishArticleUseCase.kt
│   │       └── SearchArticlesUseCase.kt
│   │
│   ├── presentation/
│   │   ├── article/
│   │   │   ├── list/
│   │   │   │   ├── ArticleListScreen.kt
│   │   │   │   └── ArticleListViewModel.kt
│   │   │   └── detail/
│   │   │       ├── ArticleDetailScreen.kt
│   │   │       └── ArticleDetailViewModel.kt
│   │   └── common/
│   │       ├── components/
│   │       └── theme/
│   │
│   └── di/
│       ├── DatabaseModule.kt
│       ├── NetworkModule.kt
│       └── RepositoryModule.kt
```

---

## แบบฝึกหัด Part 51

```kotlin
// แบบฝึกหัด: สร้าง Clean Architecture สำหรับ Weather App

// Domain Models
data class WeatherCondition(
    val city: String,
    val temperature: Double,
    val feelsLike: Double,
    val humidity: Int,
    val condition: String,
    val windSpeed: Double,
    val icon: String
)

data class ForecastDay(
    val date: LocalDate,
    val high: Double,
    val low: Double,
    val condition: String
)

// TODO: สร้าง Domain Layer:
// - WeatherRepository interface
// - GetCurrentWeatherUseCase
// - GetForecastUseCase
// - GetFavoriteCitiesUseCase
// - AddFavoriteCityUseCase

// TODO: สร้าง Data Layer:
// - WeatherApiService (Retrofit)
// - WeatherDto, ForecastDto
// - Mappers
// - WeatherLocalDataSource (Room)
// - WeatherRepositoryImpl

// TODO: สร้าง Presentation Layer:
// - WeatherViewModel (ใช้ Use Cases)
// - WeatherScreen (Compose)
// - UiState, UiEvents
```

---

*Part 51 จบแล้ว | ก่อนหน้า: [Part 30](../part30/README.md) | ถัดไป: [Part 52](../part52/README.md)*
