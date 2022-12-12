---
categories: [Android]
tags: [retrofit, kotlin coroutines]
---

본 포스팅에서는 Retrofit의 Call Adapter를 커스텀하여, 내가 원하는 상태처리를 해보도록 하겠습니다.

Api는 Unsplash의 api를 이용하겠습니다.

Retrofit에서 받을 수 있는 데이터의 타입은 다음과 같습니다.
- Call
- Response
- 일반 타입

여기서, Call Adapter를 커스텀하면, Call 타입을 원하는 타입으로 바꾸어 받을 수 있습니다.

이 방법을 이용하여, Android 에서 흔히 사용하는 sealed class를 이용한 상태 처리를 해보도록 하겠습니다.

먼저, 아래와 같이 상태 처리를 도와주는 sealed class를 만들어 줍니다.
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

Api를 통해 받을 데이터 클래스를 만들어줍니다.
```kotlin
data class UnsplashPhoto(
    val id: String,
    val description: String?,
    val urls: UnsplashPhotoUrls,
    val user: UnsplashUser
)
```
{: .nolineno }

이제 본격적으로 Retrofit의 Call Adapter를 커스텀해보도록 하겠습니다.

다음과 같은 순서를 거쳐 커스텀한 Call Adapter를 Retrofit에 적용시키겠습니다.
> 
1. Custom Call Class 만들기
2. Custom Call Adapter, Call Adapter Factory 만들기
3. Retrofit Builder에 추가
4. Retrofit Api Interface에 suspend keyword 적용


## Custom Call Class
Retrofit의 Custom Call Adpater의 초입입니다. 어렵지 않으니 코드를 보면 금방 이해하실 수 있습니다.

```kotlin
class ResponseCall<T> constructor(
    private val callDelegate: Call<T>
) : Call<Result<T>> {

    override fun enqueue(callback: Callback<Result<T>>) = callDelegate.enqueue(object : Callback<T> {
        override fun onResponse(call: Call<T>, response: Response<T>) {
            response.body()?.let {
                when(response.code()) {
                    in 200..299 -> {
                        callback.onResponse(this@ResponseCall, Response.success(Result.Success(it, response.code())))
                    }
                    in 400..409 -> {
                        callback.onResponse(this@ResponseCall, Response.success(Result.ApiError(response.message(), response.code())))
                    }
                }
            }?: callback.onResponse(this@ResponseCall, Response.success(Result.NullResult()))
        }

        override fun onFailure(call: Call<T>, t: Throwable) {
            callback.onResponse(this@ResponseCall, Response.success(Result.NetworkError(t)))
            call.cancel()
        }
    })

    override fun clone(): Call<Result<T>> = ResponseCall(callDelegate.clone())

    override fun execute(): Response<Result<T>> = throw UnsupportedOperationException("ResponseCall does not support execute.")

    override fun isExecuted(): Boolean = callDelegate.isExecuted

    override fun cancel() = callDelegate.cancel()

    override fun isCanceled(): Boolean = callDelegate.isCanceled

    override fun request(): Request = callDelegate.request()

    override fun timeout(): Timeout = callDelegate.timeout()
}
```
{: .nolineno }

실질적으로 우리가 받을 데이터들을 정의하는 곳입니다.

위에서 정의한 sealed class를 토대로 입맛에 맞게 데이터를 정의하여 callback.onResponse().. 해주시면 됩니다.

## Custom Call Adapter
```kotlin
class ResponseAdapter<T>(
    private val successType : Type
) : CallAdapter<T, Call<Result<T>>> {
    override fun responseType(): Type = successType

    override fun adapt(call: Call<T>): Call<Result<T>> = ResponseCall(call)
}
```
{: .nolineno }

## Custom Call AdapterFactory
```kotlin
class ResponseAdapterFactory : CallAdapter.Factory() {
    override fun get(returnType: Type, annotations: Array<out Annotation>, retrofit: Retrofit): CallAdapter<*, *>? {

        if (Call::class.java != getRawType(returnType)) return null
        check(returnType is ParameterizedType)

        val responseType = getParameterUpperBound(0, returnType)
        if (getRawType(responseType) != Result::class.java) return null
        check(responseType is ParameterizedType)


        val successType = getParameterUpperBound(0, responseType)

        return ResponseAdapter<Any>(successType)
    }
}
```
{: .nolineno }

데이터의 타입을 구분해주는 코드입니다. 

Retrofit Api Interface에서 suspend keyword를 적용해주지 않을 시, 맨 처음 returnType에서 Call Type으로 들어오지 않아 온전한 데이터가 return 되지 않습니다.

## Retrofit Builder
```kotlin
Retrofit.Builder()
	.addCallAdapterFactory(ResponseAdapterFactory())
    	.baseUrl("https://api.unsplash.com/")
        .build()
```
{: .nolineno }

일반적인 Retrofit의 Builder 형태입니다.

입맛에 맞게 HttpClient, ConverterFactory, Interceptor... 를 추가해주시면 됩니다.
## Suspend Function
```kotlin
interface RetrofitApi {

    @Headers("Accept-Version: v1", "Authorization: Client-ID --> Client ID 값 <--")
    @GET("search/photos")
    suspend fun searchPhotos(
        @Query("query") query: String,
        @Query("page") page: Int,
        @Query("per_page") perPage: Int
    ): Result<UnsplashResponse>
}
```
{: .nolineno }

**suspend**를 붙이는게 중요합니다. suspend keyword를 적용하지 않으면 api 호출 시, Call Type으로 받지 않아 온전한 데이터를 얻지 못 합니다.

## 사용법
다음과 같이 사용하면 됩니다.
```kotlin
val retrofit = Retrofit.Builder()....
val api = retrofit.create(RetrofitApi::class.java)

val response = api.searchPhotos(...)
when(response) {
    is Result.Success -> 
    is Result.ApiError ->
    is Result.NetworkError ->
    is Result.NullResult ->
}
```
{: .nolineno }

전체 코드 : https://github.com/TaehoonLeee/multi-module-clean-architecture

---
## References

1. https://proandroiddev.com/retrofit-calladapter-for-either-type-2145781e1c20

2. https://proandroiddev.com/create-retrofit-calladapter-for-coroutines-to-handle-response-as-states-c102440de37a

3. https://blogdeveloperspot.blogspot.com/2017/11/retrofit-20-custom-featrealm-gson.html