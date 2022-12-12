---
categories: [Android, Ktor]
tags: [clean architecture, di, hilt, ktor client]
---

# Ktor Client를 이용한 Android http 통신

이전 포스팅에서는 Clean Architecture 기반의 프로젝트를 구성하고 Hilt를 이용하여 Dependency Injection을 진행해주었었습니다.

이번 포스팅에서는 Ktor를 이용하여 네트워크 통신을 하겠습니다.

Ktor란, 코틀린과 마찬가지로 JetBrains사에서 개발된 코루틴 기반의 프레임워크로써 멀티플랫폼 용도로 개발되었습니다. Ktor를 사용하면 코루틴 기반의 Http Client 개발이 가능합니다.

들어가기에 앞서, Ktor는 Engine과 Serializer라는 두 요소를 가집니다. Engine을 통해 멀티플랫폼에서 원하는 dependency를 추가할 수 있고, Serialization을 통해 session data를 직렬화할 수 있습니다.

본 포스팅에서는 Android에서 사용하는 OkHttp 엔진, Gson Serialzer를 사용하겠습니다.

## Add Dependency

먼저, build.gradle(Data 모듈단)에서 dependency를 추가합니다. (본 포스팅에서는 1.6.0 버전을 사용했습니다.)
```kotlin
implementation "io.ktor:ktor-client-gson:$ktor_version"
implementation "io.ktor:ktor-client-okhttp:$ktor_version"
implementation "io.ktor:ktor-client-logging-jvm:$ktor_version"
```
{: .nolineno }

## Create HttpClient
```kotlin
@Singleton
@Provides
fun provideKtorHttpClient(): HttpClient {
    return HttpClient(OkHttp) {
    	defaultRequest {
            headers {
                append("Accept-Version", "v1")
                append(HttpHeaders.Authorization, "Client-ID id 값")
            }
            url {
                protocol = URLProtocol.HTTPS
                host = "api.unsplash.com"
            }
        }
        install(JsonFeature) {
            GsonSerializer()
        }
        install(Logging) {
            logger = Logger.DEFAULT
            level = LogLevel.ALL
        }
    }
}
```
{: .nolineno }

#### OkHttp Engine
앞서 말씀드린 대로, Ktor Client를 구성할 때 Engine을 설정할 수 있습니다. Android Platform에서는 **Android, OkHttp...**를 사용할 수 있습니다.
#### Default Request (optional)
기본 설정들을 해줍니다. ex) Base Url, Headers, ...
#### Serializer
Gson 라이브러리를 이용하지 않을 경우 **GsonSerializer** 대신 **KotlinxSerializer**를 사용하면 됩니다.
#### Logging (optional)
Network request 중 log를 남길 수 있습니다.

## Serialization
```kotlin
data class UnsplashResponse(
    val results: List<UnsplashPhoto>,
    @SerializedName("total_pages")
    val totalPages: Int
)
```
{: .nolineno }

Gson 라이브러리의 직렬화 규칙을 적용하면 됩니다.

## Configure Network Request
```kotlin
@Singleton
class KtorUnsplashService @Inject constructor(
    private val httpClient: HttpClient
) : UnsplashService {

    override suspend fun searchPhotos(
        query: String,
        page: Int,
        perPage: Int
    ): Result<UnsplashResponse> {
        return try {
            val response = httpClient.get<HttpResponse>(path = "search/photos") {
                parameter("query", query)
                parameter("page", page)
                parameter("per_page", perPage)
            }

            if (response.status.isSuccess()) {
                Result.Success(response.receive(), response.status.value)
            } else {
                Result.ApiError(response.status.description, response.status.value)
            }
        } catch (e: Exception) {
            Result.NetworkError(e)
        }
    }
}
```
{: .nolineno }

#### Hilt Annotation
HttpClient를 주입시키기 위해 **Inject Annotation** 을 적용합니다.
#### HttpResponse
저는 response의 parameters를 받기 위해 HttpResponse로 데이터를 받았습니다. (https://ktor.io/docs/response.html#parameters)

## Dependency Injection with Hilt
#### Module
```kotlin
@Module
@InstallIn(SingletonComponent::class)
object DataModule {

    @Singleton
    @Provides
    fun provideKtorHttpClient(): HttpClient {
        return HttpClient(OkHttp) {
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

#### HttpClient
[Create HttpClient](#Create-HttpClient)와 같이 설정을 해줍니다.

#### UnsplashRepository
```kotlin
interface UnsplashRepository {
    fun getSearchResultOfPage(query: String, page: Int): Flow<Result<List<UnsplashPhoto>>>

    // PagingData를 사용하기 위한 Generic Function
    fun <T> getSearchResult(query: String): Flow<T>
}
```
{: .nolineno }

UnsplashService와 마찬가지로 인터페이스를 따로 만들어 KtorUnsplashRepository가 **UnsplasRepository를 implement** 합니다.

코드 : <https://github.com/TaehoonLeee/multi-module-clean-architecture>


---
## References

1. <https://medium.com/google-developer-experts/how-to-use-ktor-client-on-android-dcdeddc066b9>
2. <https://medium.com/l-r-engineering/migrating-retrofit-to-ktor-93bdaf58d7d4>
3. <https://ktor.io/docs/welcome.html>