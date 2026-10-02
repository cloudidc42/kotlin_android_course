# Part 50: Android - Paging 3 พื้นฐาน
## ขั้นตอนที่ 1026-1050

---

## ขั้นตอนที่ 1026: Paging 3 คืออะไร

Paging 3 ช่วยโหลดข้อมูลแบบ pagination (โหลดทีละหน้า) ประหยัด memory และ bandwidth

```kotlin
// build.gradle.kts
dependencies {
    val pagingVersion = "3.2.1"
    implementation("androidx.paging:paging-runtime:$pagingVersion")
    implementation("androidx.paging:paging-compose:$pagingVersion")

    // Room Paging
    implementation("androidx.room:room-paging:2.6.1")
}

// ส่วนประกอบหลักของ Paging 3:
// 1. PagingSource  - แหล่งข้อมูล (API/DB)
// 2. PagingConfig  - ตั้งค่า page size
// 3. Pager         - สร้าง PagingData stream
// 4. PagingDataAdapter (XML) / LazyPagingItems (Compose)
```

---

## ขั้นตอนที่ 1027: สร้าง PagingSource จาก API

```kotlin
import androidx.paging.PagingSource
import androidx.paging.PagingState

data class Movie(
    val id: Int,
    val title: String,
    val overview: String,
    val posterPath: String,
    val rating: Float,
    val releaseDate: String
)

data class MoviesResponse(
    val page: Int,
    val results: List<Movie>,
    val totalPages: Int,
    val totalResults: Int
)

interface MovieApi {
    @GET("movie/popular")
    suspend fun getPopularMovies(
        @Query("page") page: Int,
        @Query("language") language: String = "th-TH"
    ): MoviesResponse
}

// PagingSource - ดึงข้อมูลทีละหน้า
class MoviePagingSource(
    private val api: MovieApi
) : PagingSource<Int, Movie>() {

    override suspend fun load(params: LoadParams<Int>): LoadResult<Int, Movie> {
        // หน้าปัจจุบัน (เริ่มที่ 1)
        val currentPage = params.key ?: 1

        return try {
            val response = api.getPopularMovies(page = currentPage)
            val movies = response.results

            LoadResult.Page(
                data = movies,
                prevKey = if (currentPage == 1) null else currentPage - 1,
                nextKey = if (currentPage < response.totalPages) currentPage + 1 else null
            )
        } catch (e: IOException) {
            LoadResult.Error(e)
        } catch (e: HttpException) {
            LoadResult.Error(e)
        }
    }

    // เมื่อ Pager invalidate (refresh) จะเริ่มจากหน้าไหน
    override fun getRefreshKey(state: PagingState<Int, Movie>): Int? {
        return state.anchorPosition?.let { anchorPosition ->
            state.closestPageToPosition(anchorPosition)?.prevKey?.plus(1)
                ?: state.closestPageToPosition(anchorPosition)?.nextKey?.minus(1)
        }
    }
}
```

---

## ขั้นตอนที่ 1028: Repository กับ Pager

```kotlin
import androidx.paging.Pager
import androidx.paging.PagingConfig
import androidx.paging.PagingData
import kotlinx.coroutines.flow.Flow

class MovieRepository(
    private val api: MovieApi,
    private val movieDao: MovieDao // สำหรับ RemoteMediator
) {
    // PagingData Flow จาก API
    fun getPopularMovies(): Flow<PagingData<Movie>> {
        return Pager(
            config = PagingConfig(
                pageSize = 20,            // จำนวน item ต่อหน้า
                prefetchDistance = 5,     // โหลดล่วงหน้าเมื่อเหลือ 5 items
                enablePlaceholders = false,
                initialLoadSize = 40      // โหลดครั้งแรก 2 หน้า
            ),
            pagingSourceFactory = { MoviePagingSource(api) }
        ).flow
    }

    // สำหรับ Room Paging
    fun getLocalMovies(): Flow<PagingData<Movie>> {
        return Pager(
            config = PagingConfig(pageSize = 20),
            pagingSourceFactory = { movieDao.getMoviesPaged() }
        ).flow
    }
}

// Room Dao สำหรับ Paging
@Dao
interface MovieDao {
    // Room จัดการ PagingSource อัตโนมัติ
    @Query("SELECT * FROM movies ORDER BY rating DESC")
    fun getMoviesPaged(): PagingSource<Int, Movie>

    @Insert(onConflict = OnConflictStrategy.REPLACE)
    suspend fun insertMovies(movies: List<Movie>)

    @Query("DELETE FROM movies")
    suspend fun clearAll()
}
```

---

## ขั้นตอนที่ 1029: ViewModel กับ PagingData

```kotlin
import androidx.lifecycle.ViewModel
import androidx.lifecycle.viewModelScope
import androidx.paging.PagingData
import androidx.paging.cachedIn
import androidx.paging.filter
import androidx.paging.map
import kotlinx.coroutines.flow.*

class MovieViewModel(
    private val repository: MovieRepository
) : ViewModel() {

    private val _searchQuery = MutableStateFlow("")
    val searchQuery: StateFlow<String> = _searchQuery.asStateFlow()

    // cachedIn - cache PagingData ไว้ใน viewModelScope
    // ป้องกันการ reload เมื่อ configuration change
    val movies: Flow<PagingData<Movie>> = repository
        .getPopularMovies()
        .cachedIn(viewModelScope)

    // PagingData พร้อม filter
    val filteredMovies: Flow<PagingData<Movie>> = combine(
        repository.getPopularMovies().cachedIn(viewModelScope),
        _searchQuery
    ) { pagingData, query ->
        if (query.isEmpty()) {
            pagingData
        } else {
            pagingData.filter { movie ->
                movie.title.contains(query, ignoreCase = true)
            }
        }
    }

    // Transform PagingData
    val movieUiModels: Flow<PagingData<MovieUiModel>> = repository
        .getPopularMovies()
        .map { pagingData ->
            pagingData.map { movie ->
                MovieUiModel(
                    id = movie.id,
                    title = movie.title,
                    rating = "⭐ ${movie.rating}",
                    posterUrl = "https://image.tmdb.org/t/p/w500${movie.posterPath}"
                )
            }
        }
        .cachedIn(viewModelScope)

    fun setSearchQuery(query: String) {
        _searchQuery.value = query
    }
}

data class MovieUiModel(
    val id: Int,
    val title: String,
    val rating: String,
    val posterUrl: String
)
```

---

## ขั้นตอนที่ 1030: LazyPagingItems ใน Compose

```kotlin
import androidx.compose.foundation.lazy.grid.LazyVerticalGrid
import androidx.compose.foundation.lazy.grid.GridCells
import androidx.paging.LoadState
import androidx.paging.compose.collectAsLazyPagingItems
import androidx.paging.compose.itemKey

@Composable
fun MovieListScreen(viewModel: MovieViewModel = hiltViewModel()) {
    val movies = viewModel.movies.collectAsLazyPagingItems()

    // Pull to refresh
    val pullRefreshState = rememberPullToRefreshState()
    if (pullRefreshState.isRefreshing) {
        LaunchedEffect(true) {
            movies.refresh()
            pullRefreshState.endRefresh()
        }
    }

    Box(Modifier.fillMaxSize().nestedScroll(pullRefreshState.nestedScrollConnection)) {
        LazyVerticalGrid(
            columns = GridCells.Fixed(2),
            contentPadding = PaddingValues(8.dp),
            horizontalArrangement = Arrangement.spacedBy(8.dp),
            verticalArrangement = Arrangement.spacedBy(8.dp)
        ) {
            // Header
            item(span = { GridItemSpan(maxLineSpan) }) {
                when (movies.loadState.refresh) {
                    is LoadState.Loading -> {
                        Box(
                            modifier = Modifier.fillMaxWidth().height(200.dp),
                            contentAlignment = Alignment.Center
                        ) { CircularProgressIndicator() }
                    }
                    is LoadState.Error -> {
                        val error = movies.loadState.refresh as LoadState.Error
                        Column(
                            modifier = Modifier.fillMaxWidth().padding(16.dp),
                            horizontalAlignment = Alignment.CenterHorizontally
                        ) {
                            Text(
                                "โหลดไม่ได้: ${error.error.message}",
                                color = MaterialTheme.colorScheme.error
                            )
                            Button(onClick = { movies.retry() }) {
                                Text("ลองใหม่")
                            }
                        }
                    }
                    else -> { /* แสดงรายการปกติ */ }
                }
            }

            // Movie items
            items(
                count = movies.itemCount,
                key = movies.itemKey { it.id }
            ) { index ->
                val movie = movies[index]
                if (movie != null) {
                    MovieCard(movie = movie)
                } else {
                    // Placeholder ขณะโหลด
                    MovieCardPlaceholder()
                }
            }

            // Footer - loading more
            item(span = { GridItemSpan(maxLineSpan) }) {
                when (movies.loadState.append) {
                    is LoadState.Loading -> {
                        Box(
                            modifier = Modifier.fillMaxWidth().padding(8.dp),
                            contentAlignment = Alignment.Center
                        ) { CircularProgressIndicator(modifier = Modifier.size(32.dp)) }
                    }
                    is LoadState.Error -> {
                        Row(
                            modifier = Modifier.fillMaxWidth().padding(8.dp),
                            horizontalArrangement = Arrangement.Center
                        ) {
                            Text("โหลดเพิ่มไม่ได้")
                            TextButton(onClick = { movies.retry() }) {
                                Text("ลองใหม่")
                            }
                        }
                    }
                    else -> {}
                }
            }
        }

        PullToRefreshContainer(
            state = pullRefreshState,
            modifier = Modifier.align(Alignment.TopCenter)
        )
    }
}

@Composable
fun MovieCard(movie: Movie) {
    Card(modifier = Modifier.fillMaxWidth()) {
        Column {
            AsyncImage(
                model = "https://image.tmdb.org/t/p/w500${movie.posterPath}",
                contentDescription = movie.title,
                modifier = Modifier
                    .fillMaxWidth()
                    .aspectRatio(2f / 3f),
                contentScale = ContentScale.Crop
            )
            Column(modifier = Modifier.padding(8.dp)) {
                Text(
                    text = movie.title,
                    style = MaterialTheme.typography.bodyMedium,
                    maxLines = 2,
                    overflow = TextOverflow.Ellipsis
                )
                Text(
                    text = "⭐ ${movie.rating}",
                    style = MaterialTheme.typography.bodySmall,
                    color = MaterialTheme.colorScheme.primary
                )
            }
        }
    }
}

@Composable
fun MovieCardPlaceholder() {
    Card(modifier = Modifier.fillMaxWidth()) {
        Column {
            Box(
                modifier = Modifier
                    .fillMaxWidth()
                    .aspectRatio(2f / 3f)
                    .background(Color.LightGray)
            )
            Box(modifier = Modifier.padding(8.dp)) {
                Box(
                    modifier = Modifier
                        .fillMaxWidth()
                        .height(16.dp)
                        .background(Color.LightGray, RoundedCornerShape(4.dp))
                )
            }
        }
    }
}
```

---

## ขั้นตอนที่ 1031: RemoteMediator - Paging กับ Local Cache

```kotlin
import androidx.paging.ExperimentalPagingApi
import androidx.paging.RemoteMediator
import androidx.room.withTransaction

// RemoteMediator - ดึงจาก API และเก็บลง Room
@OptIn(ExperimentalPagingApi::class)
class MovieRemoteMediator(
    private val api: MovieApi,
    private val database: AppDatabase
) : RemoteMediator<Int, Movie>() {

    override suspend fun initialize(): InitializeAction {
        // ตรวจสอบว่าควร refresh หรือไม่
        val cacheTimeout = 30 * 60 * 1000L // 30 นาที
        val lastUpdated = database.remoteKeysDao().getLastUpdated() ?: 0
        return if (System.currentTimeMillis() - lastUpdated > cacheTimeout) {
            InitializeAction.LAUNCH_INITIAL_REFRESH
        } else {
            InitializeAction.SKIP_INITIAL_REFRESH
        }
    }

    override suspend fun load(
        loadType: LoadType,
        state: PagingState<Int, Movie>
    ): MediatorResult {
        val page = when (loadType) {
            LoadType.REFRESH -> 1
            LoadType.PREPEND -> return MediatorResult.Success(endOfPaginationReached = true)
            LoadType.APPEND -> {
                val remoteKey = database.remoteKeysDao().getLastKey()
                remoteKey?.nextPage ?: return MediatorResult.Success(endOfPaginationReached = true)
            }
        }

        return try {
            val response = api.getPopularMovies(page)
            val endOfPagination = response.page >= response.totalPages

            database.withTransaction {
                if (loadType == LoadType.REFRESH) {
                    database.movieDao().clearAll()
                    database.remoteKeysDao().clearAll()
                }
                database.movieDao().insertMovies(response.results)
                database.remoteKeysDao().insert(
                    RemoteKey(nextPage = if (endOfPagination) null else page + 1)
                )
            }

            MediatorResult.Success(endOfPaginationReached = endOfPagination)
        } catch (e: IOException) {
            MediatorResult.Error(e)
        } catch (e: HttpException) {
            MediatorResult.Error(e)
        }
    }
}

// ใช้ RemoteMediator ใน Pager
@OptIn(ExperimentalPagingApi::class)
fun getMoviesWithRemoteMediator(
    api: MovieApi,
    database: AppDatabase
): Flow<PagingData<Movie>> {
    return Pager(
        config = PagingConfig(pageSize = 20),
        remoteMediator = MovieRemoteMediator(api, database),
        pagingSourceFactory = { database.movieDao().getMoviesPaged() }
    ).flow
}
```

---

*Part 50 จบแล้ว | ก่อนหน้า: [Part 49](../part49/README.md) | ถัดไป: [Part 51](../part51/README.md)*
