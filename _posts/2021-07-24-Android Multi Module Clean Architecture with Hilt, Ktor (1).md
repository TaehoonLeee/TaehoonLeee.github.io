---
categories: [Android, Architecture]
tags: [clean architecture, di, hilt, ktor client]
---

# 프로젝트 개요
Clean Architecture 기반의 간단한 프로젝트를 만들어보겠습니다.
**Hilt**를 이용한 Dependency Injection, **Ktor Client**를 이용한 Http 통신을 하겠습니다.
https://unsplash.com 의 api를 사용하겠습니다.

본 포스팅에서는 아래와 같은 내용을 다루도록 하겠습니다.
1. Clean Architecutre 적용
2. Hilt를 이용한 DI 적용

## Clean Architecutre 적용
![](https://images.velog.io/images/ams770/post/e44df0b9-8418-4011-9c23-e1d07b2ff46a/clean-architecture-own-layers.png)<sup>[1](#footnote_1)</sup>
Clean Architecutre 라고 하면 가장 먼저 떠오르는 원 이미지입니다.
위의 구조를 우리는 **Presentation Layer에 MVVM 패턴을 적용**시켜 다음과 같은 구조를 띄게 하겠습니다.
![](https://images.velog.io/images/ams770/post/b0d08f81-7e5b-4d4a-9634-191f89bf0c97/1*MzkbfQsYb0wTBFeqplRoKg.png)<sup>[2](#footnote_2)</sup>

따라서, Presentation Module, Domain Module, Data Module 총 세 가지 모듈을 프로젝트에 적용하겠습니다.

## Domain Module (Kotlin pure library)
### Add Dependency
```kotlin
implementation "org.jetbrains.kotlinx:kotlinx-coroutines-core:$coroutine_core_version"
implementation "androidx.paging:paging-common:${paging_version}"
implementation group: 'javax.inject', name: 'javax.inject', version: '1'
```
{: .nolineno }

먼저, Domain Module부터 보겠습니다.
Domain Module에서 생성해야 할 것들은 크게 세 가지가 있습니다.
#### Repository Interface
Data Layer에서 사용할 Repository의 Abstraction 입니다.
왜, 이런 Abstraction이 필요하냐 함은 SOLID의 원칙 중 하나인 **DIP(Dependency Inversion Principle)**을 충족시키기 위함입니다. 이와 관련한 내용은 좋은 자료가 많으니 더 자세히 다루지는 않겠습니다.

코드를 보겠습니다.
```kotlin
interface UnsplashRepository {
    fun getSearchResultOfPage(query: String, page: Int): Flow<Result<List<UnsplashPhoto>>>

    fun getSearchResult(query: String): Flow<PagingData<UnsplashPhoto>
}
```
{: .nolineno }

PagingData가 Android 종속적이지만, domain module에서 androidx paging common 라이브러리를 주입해주었기 때문에 사용이 가능합니다.

#### Use Case
```kotlin
@Singleton
class GetSearchResultUseCase @Inject constructor(
    private val unsplashRepository: UnsplashRepository
) {

    operator fun invoke(query: String) = unsplashRepository.getSearchResult(query)
}
```
{: .nolineno }

실제 사용자가 하는 일련의 행동들을 나타냅니다.
**JSR-330의 Inject Annotation**을 이용하여 Repository를 주입시켜 줍니다.
여기에 대한 내용은 [이 포스팅](https://velog.io/@ams770/Hilt와-Dagger-JSR-330-3-Pure-JavaKotlin-Module)을 참고해주시기 바랍니다.

#### Entity
```kotlin
data class UnsplashPhoto(
    val id: String,
    val description: String?,
    val urls: UnsplashPhotoUrls,
    val user: UnsplashUser
) {

    data class UnsplashPhotoUrls(
        val raw: String,
        val full: String,
        val regular: String,
        val small: String,
        val thumb: String,
    )

    data class UnsplashUser(
        val name: String,
        val username: String
    )
}
```
{: .nolineno }

```kotlin
sealed class Result<T> {

    class Success<T>(val data: T, val code: Int) : Result<T>()

    class Loading<T> : Result<T>()

    class ApiError<T>(val message: String, val code: Int) : Result<T>()

    class NetworkError<T>(val throwable: Throwable) : Result<T>()

    class NullResult<T> : Result<T>()
}
```
{: .nolineno }

상태 관리를 위한 Result 클래스와 사용할 데이터 객체를 정의해줍니다.

## Data Module (Android Library)
### Add Dependency
```kotlin
implementation "org.jetbrains.kotlinx:kotlinx-coroutines-core:$coroutine_core_version"
implementation "org.jetbrains.kotlinx:kotlinx-coroutines-android:$coroutine_android_version"

implementation "com.google.dagger:hilt-android:$hilt_version"
kapt "com.google.dagger:hilt-android-compiler:$hilt_version"

implementation "com.google.code.gson:gson:$gson_version"

implementation "androidx.paging:paging-runtime-ktx:$paging_version"
```
{: .nolineno }

데이터 모듈에서 생성해야 할 것들은 크게 다음과 같습니다.
#### Repository Implementations
```kotlin
class KtorUnsplashRepositoryImpl @Inject constructor(
    private val unsplashService: UnsplashService
) : UnsplashRepository {

    override fun getSearchResultOfPage(
        query: String,
        page: Int
    ): Flow<Result<List<UnsplashPhoto>>> = flow {
        val response = unsplashService.searchPhotos(query, page)
        emit(ResponseMapper.responseToPhotoList(response))
    }

    override fun getSearchResult(query: String) =
        Pager(
            config = PagingConfig(
                pageSize = 20,
                maxSize = 100,
                enablePlaceholders = false
            ),
            pagingSourceFactory = { UnsplashPhotoPagingSource(unsplashService, query) }
        ).flow
}
```
{: .nolineno }

```kotlin
class UnsplashPagingSource constructor(
    private val unsplashService: UnsplashService,
    private val query: String
) : PagingSource<Int, UnsplashPhoto>() {
    override fun getRefreshKey(state: PagingState<Int, UnsplashPhoto>): Int? {
        return state.anchorPosition
    }

    override suspend fun load(params: LoadParams<Int>): LoadResult<Int, UnsplashPhoto> {
        val position = params.key?: 1
        val response = unsplashService.searchPhotos(query, position, params.loadSize)

        return try {
            response as Result.Success
            LoadResult.Page(
                data = response.data.results,
                prevKey = if (position == 1) null else position - 1,
                nextKey = if (position == response.data.totalPages) null else position + 1
            )
        } catch (e: Exception) {
            LoadResult.Error(e)
        }
    }
}
```
{: .nolineno }

#### Entity
```kotlin
data class UnsplashResponse(
    val results: List<UnsplashPhoto>,
    @SerializedName("total_pages")
    val totalPages: Int
)
```
{: .nolineno }

Gson 라이브러리의 직렬화 규칙을 사용해줍니다.
#### Mapper
```kotlin
object ResponseMapper {

    fun responseToPhotoList(response: Result<UnsplashResponse>): Result<List<UnsplashPhoto>> {
        return when(response) {
            is Result.Success -> Result.Success(response.data.results, response.code)
            is Result.ApiError -> Result.ApiError(response.message, response.code)
            is Result.NetworkError -> Result.NetworkError(response.throwable)
            is Result.NullResult -> Result.NullResult()
            is Result.Loading -> Result.Loading()
        }
    }
}
```
{: .nolineno }

Data Source로 부터 받은 데이터를 Domain Layer의 데이터 객체로 맵핑해줍니다.
#### Module (optional Hilt)
```kotlin
@Module
@InstallIn(SingletonComponent::class)
object DataModule {

    @Singleton
    @Provides
    fun provideKtorHttpClient(): HttpClient {
        return HttpClient(OkHttp) {
			// 2.0 이상
			install(ContentNegotiation) {
				gson()
			}
			// 2.0 이하
            install(JsonFeature) {
                GsonSerializer()
            }
            install(Logging) {
                logger = Logger.DEFAULT
                level = LogLevel.ALL
            }
        }
    }
}

@Module
@InstallIn(SingletonComponent::class)
interface RepositoryModule {

    @Binds
    @Singleton
    fun bindUnsplashService(ktorUnsplashService: KtorUnsplashService): UnsplashService

    @Binds
    @Singleton
    fun bindUnsplashRepository(unsplashRepositoryImpl: KtorUnsplashRepositoryImpl): UnsplashRepository
}
```
{: .nolineno }

Ktor Client를 생성하는 내용은 추후에 다루도록 하겠습니다.
#### Data Sources
```kotlin
interface UnsplashService {

    suspend fun searchPhotos(query: String, page: Int, perPage: Int): Result<UnsplashResponse>
}
```
{: .nolineno }

Ktor Client를 이용한 Http 통신만을 사용할 예정이기 때문에 Remote Data Source 만 생성합니다.

## Presentation Layer

#### View
```kotlin
@AndroidEntryPoint
class GalleryFragment : Fragment() {
    private val galleryViewModel: GalleryViewModel by viewModels()
        
    lifecycleScope.launch {
        galleryViewModel.searchResult.flowWithLifecycle(lifecycle).collect {

        }
    }
}
```
{: .nolineno }

ViewModel의 데이터를 처리하는 부분입니다.
#### ViewModel
```kotlin
@HiltViewModel
class GalleryViewModel @Inject constructor(
	getSearchResult: GetSearchResultUseCase,
) : ViewModel() {

	val searchResult = getSearchResult(DEFAULT_QUERY)
		.cachedIn(viewModelScope)

	companion object {
		const val DEFAULT_QUERY = "cats"
	}
}
```
{: .nolineno }

Domain Layer의 Use Case를 이용해 원하는 데이터를 받는 코드입니다.

## End
다음 포스팅에서는 Ktor Client를 생성하여 프로젝트에 적용하겠습니다.

코드 : <https://github.com/TaehoonLeee/multi-module-clean-architecture>

--- 
## References

<a name="footnote_1">1</a>:  <https://antonioleiva.com/clean-architecture-android/>

<a name="footnote_2">2</a>: <https://tech.olx.com/clean-architecture-and-mvvm-on-ios-c9d167d9f5b3>