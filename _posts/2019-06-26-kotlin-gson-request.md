---
layout: post
title:  "[Kotlin] Volley Gson Reqeust"
author: haejung
categories: [ kotlin, gson, volley ]
image: assets/images/kotlin_android_logo.png
featured: true
hidden: true
---



### Volley GSON Reqeust 

> 주로 Retrofit2를 사용하지만 간혹 Okhttp3가 아닌 Custom Http Client를 사용하기 위하여 Volley를 사용하곤 합니다. Volley는 Request를 상속하여 원하는 형태의 Request 를 작성하도록 되어 있습니다. DTO를 Serialize/Deserialze하기 위한 도구인 GSON을 이용한 Request를 간단하게 만들어 봤습니다.



##### 적용 환경

* Kotlin
* Volley



* 파일: GsonRequest.kt

```kotlin
import com.android.volley.NetworkResponse
import com.android.volley.ParseError
import com.android.volley.Request
import com.android.volley.Response
import com.android.volley.toolbox.HttpHeaderParser
import com.google.gson.Gson
import com.google.gson.JsonSyntaxException
import com.google.gson.reflect.TypeToken
import java.io.UnsupportedEncodingException
import java.lang.reflect.Type
import java.nio.charset.Charset

/**
 * GsonRequest with general method
 */
inline fun <reified T> gsonRequest(block: GsonRequestParam<T>.() -> Unit) =
    GsonRequest(GsonRequestParam<T>().apply(block), object : TypeToken<T>() {}.type)

/**
 * GsonRequest with post method
 */
inline fun <reified T> gsonPostRequest(block: GsonRequestParam<T>.() -> Unit) =
    GsonRequest(
        GsonRequestParam<T>(method = Request.Method.POST).apply(block),
        object : TypeToken<T>() {}.type
    )

/**
 * GsonRequestParam for building request
 * method: GET, POST, PUT, PATCH, DELETE
 * url: URL address
 * headers: optional
 * query: optional (Only for GET method)
 * body: optional (for POST, PUT, PATCH methods)
 * listener: listener for result
 * error: error listener
 */
data class GsonRequestParam<T>(
    var method: Int = Request.Method.GET,
    var url: String = "",
    var headers: MutableMap<String, String>? = null,
    var query: MutableMap<String, Any>? = null,
    var body: Any? = null,
    var listener: Response.Listener<T> =
        Response.Listener { TODO("You should implement response listener") },
    var error: Response.ErrorListener =
        Response.ErrorListener { TODO("You should implement error listener") }
) {
    fun getEncodedUrl(): String {
        if (method == Request.Method.GET) {
            query?.let { query ->
                val transformedQuery = query.map {
                    it.key.plus("=").plus(it.value)
                }.joinToString("&")
                url = url.plus("?").plus(transformedQuery)
            }
        }
        return url
    }
}

class GsonRequest<T>(
    private val params: GsonRequestParam<T>,
    private val type: Type
) : Request<T>(params.method, params.getEncodedUrl(), params.error) {

    private val gson = Gson()

    override fun parseNetworkResponse(response: NetworkResponse?): Response<T> {
        return try {
            val json = String(
                response?.data ?: ByteArray(0),
                Charset.forName(HttpHeaderParser.parseCharset(response?.headers))
            )
            Response.success(
                gson.fromJson(json, type),
                HttpHeaderParser.parseCacheHeaders(response)
            )
        } catch (e: UnsupportedEncodingException) {
            Response.error(ParseError(e))
        } catch (e: JsonSyntaxException) {
            Response.error(ParseError(e))
        }
    }

    override fun deliverResponse(response: T) = params.listener.onResponse(response)

    override fun getHeaders(): MutableMap<String, String> = params.headers ?: super.getHeaders()

    override fun getPostBodyContentType() = bodyContentType

    override fun getPostBody() = body

    override fun getBodyContentType() = "application/json"

    override fun getBody(): ByteArray = gson.toJson(params.body).toByteArray()

}
```



* DTOs (Request/Response)

```kotlin
// Login Request
data class TestLoginRequest(val userId: String, val pw: String)

// Response (성공여부와 관련 메시지, 유효 페이로드)
data class TestResponse<T>(val retCode: Int, val retMsg: String, val payload: T)

// Login Response (TestResponse에 payload에 단일/리스트로 Deserialize되는 세부 Response)
data class TestLoginResponse(val status: Int)

// User Response (TestResponse에 payload에 단일/리스트로 Deserialize되는 세부 Response)
data class TestUserResponse(
    val userId: String,
    val userName: String,
    val userEngName: String = "",
    val birthDay: String = "",
    val position: String = "",
    val center: String = "",
    val agency: String = "",
    val employTp: String = "",
    val lastLoginDt: String = "",
    val userRole: String = "",
    val regUser: String = "",
    val pwStatus: Int = 0,

    val page: Int = 0,
    val totalPage: Int = 0,
    val totalRow: Int = 0
)
```



##### 사용법

* Get

```kotlin
val requestQueue = getReqeustQueue() // Depends on your code
..
..
requestQueue.add(
    gsonRequest<TestResponse<List<TestUserResponse>>> {
        url = authSource.hostAddress.plus("/api/users")
        query = mutableMapOf(
            "page" to page + 1,
            "pageSize" to pageSize,
            "sortKey" to "raw",
            "sortDesc" to true
        )
        listener = Listener { response ->
        	response?.let {
            	when (it.retCode) {
                	0 -> callback.onUsersLoaded(
                        fromResponsesToModels(it.payload)
                    )
					else -> callback.onDataNotAvailable()
                }
            }
        }
        error = ErrorListener {
            callback.onDataNotAvailable()
        }
    }
)
```

* Post

```kotlin
val requestQueue = getReqeustQueue()
..
..
requestQueue.add(
    gsonPostRequest<TestResponse<TestLoginResponse>> {
        url = authSource.hostAddress.plus("/api/login")
        body = TestLoginRequest(id, password)
        listener = Listener { response ->
            response?.let {
                when (it.retCode) {
                    0 -> {
                        isLoggedIn = true
                        success()
                    }
                    else -> fail(it.retMsg)
                }
            }
        }
        error = ErrorListener {
            fail(it.message ?: "Failed to login")
        }
    }
)
```



#### 마무리

Gson Request에서 Type을 전달하기 위하여 inline과 reified를 사용하여 T에 대하여 구체화 함으로서 Gson에서 Deserialize하기 위한 Type정보를 정상적으로 전달하게 된다.

