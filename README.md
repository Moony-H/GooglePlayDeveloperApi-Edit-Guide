# Google Play Developer Api edits Guide

이번에는 Google Play Developer의 Edit의 사용법을 알아보겠습니다.

## 사용 환경

이번 글에서 사용할 언어는 Kotlin 입니다. compiler로는 Intellij CE를 사용합니다.

또한 Retrofit2를 이용하여 Rest API를 호출할 것입니다.

이 글을 보기 전에 앞서 설명한 [Google Play Console API Guide](https://github.com/Moony-H/GooglePlayConsoleAPIGuide)의 내용을 먼저 보시기 바랍니다.


## Edit 설명

Edit은 현재 앱의 정보와 같은 값을 가지는 객체입니다.

나중에 설명할 POST 메소드로 insert를 실행시키면 현재 앱의 정보를 복사하여 자신에게 저장합니다.

그 후 반환 값으로 앱의 모든 정보가 아닌 생성한 edit의 id와 만료 시간만 반환합니다.

앱의 정보를 알고 싶다면 이 edit의 id를 활용하여 정보를 가져올 수 있습니다.

지정된 만료 시간이 지나면 edit은 만료되어 더 이상 id를 활용하여 정보를 수정하거나 가져올 수 없습니다.

<br/>

edit은 단 한개만 생성할 수 있습니다.

만약 edit을 insert로 하나 더 생성한다면 전에 있던 edit은 만료가 되어 더이상 사용할 수 없습니다.


## Retrofit2

이번 글에서는 Retrofit2를 사용합니다. 따라서 왼쪽 Project 탭의 build.gradle 파일에 아래와 같은 코드로 Retrofit2를 추가합니다.

![스크린샷 2022-12-06 시간: 11 47 11](https://user-images.githubusercontent.com/53536205/205797038-0248a6be-1db0-4b89-a7a7-6af9216a588c.png)

```build.gradle
dependencies {
    
    //... 기존 코드
    
    //추가할 코드들
    implementation 'com.squareup.retrofit2:retrofit:2.9.0'
    implementation 'com.squareup.retrofit2:converter-gson:2.9.0'
    implementation 'com.jakewharton.retrofit:retrofit2-kotlin-coroutines-adapter:0.9.2'
    implementation 'com.squareup.okhttp3:logging-interceptor:4.10.0'

    //... 기존 코드

}
```

그 다음 오른쪽 끝의 Gradle을 누르고 Reload를 눌러 적용합니다.

![스크린샷 2022-12-06 시간: 11 57 02](https://user-images.githubusercontent.com/53536205/205798932-5528bdd7-f0bc-4e56-ba26-138735435502.png)

![스크린샷 2022-12-06 시간: 11 58 18](https://user-images.githubusercontent.com/53536205/205799196-7a8cad5f-9f7c-4324-81e5-744ea2762f2d.png)


## Edit 권한 scope 설정


Edit을 생성하기 전에, 권한 scpoe를 설정해야 합니다.

또한 다른 Repository에서 설명한 [Google Play Console API](https://github.com/Moony-H/GooglePlayConsoleAPIGuide)를 사용할 준비가 되어 있어야 합니다.

사용할 준비가 되어 있다면 아래와 같은 코드를 Main에 준비합니다.

```kotlin

import com.google.auth.oauth2.ServiceAccountCredentials
import java.io.FileInputStream

fun main(args: Array<String>) {
    val credentials =
        ServiceAccountCredentials.fromStream(FileInputStream("{Service Account Key 경로}"))


}

```

그 다음 현재 사용할 api의 권한 scope가 필요합니다. 따라서 아래와 같이 scope를 추가하고 refresh 합니다.

```kotlin
import com.google.auth.oauth2.ServiceAccountCredentials
import java.io.FileInputStream

fun main(args: Array<String>) {
    val credentials =
        ServiceAccountCredentials.fromStream(FileInputStream("{Service Account Key 경로}"))
            .createScoped(setOf("https://www.googleapis.com/auth/androidpublisher")) //edit을 사용하기 위한 권한 scope.
    credentials.refreshIfExpired()

}
```

이제 access token을 활용하여 edit에 접근할 수 있습니다.

## Edit insert

edit을 생성하는 API 입니다.

[Google Play Developer API 문서](https://developers.google.com/android-publisher/api-ref/rest/v3/edits/insert)를 참고하여 메소드와 data class를 아래와 같이 만듭니다.

**EditResponse.kt**
```kotlin
import com.google.gson.annotations.SerializedName

data class EditResponse (
    @SerializedName("id")
    val id:String,
    @SerializedName("expiryTimeSeconds")
    val expiryTimeSeconds:String
)
```

<br/>

<br/>

**EditsAPI.kt**
```kotlin
import retrofit2.Response
import retrofit2.http.Header
import retrofit2.http.POST
import retrofit2.http.Path

interface EditsAPI {

    @POST("{packageName}/edits")
    suspend fun postInsertEdit(
        @Header("Authorization")
        token: String,
        @Path("packageName")
        packageName: String,
    ): Response<EditResponse>

}
```


그 다음 Main.kt의 코드를 아래와 같이 수정합니다.

```kotlin
import com.google.auth.oauth2.ServiceAccountCredentials
import com.jakewharton.retrofit2.adapter.kotlin.coroutines.CoroutineCallAdapterFactory
import kotlinx.coroutines.*
import okhttp3.OkHttpClient
import retrofit2.Retrofit
import retrofit2.converter.gson.GsonConverterFactory
import java.io.FileInputStream
import java.util.*
import kotlin.system.exitProcess

fun main(args: Array<String>) {

    runBlocking {
        val token = getAccessToken()
        val editsRetrofit = Retrofit.Builder()
            .baseUrl("https://androidpublisher.googleapis.com/androidpublisher/v3/applications/")
            .addConverterFactory(GsonConverterFactory.create())
            .addCallAdapterFactory(CoroutineCallAdapterFactory())
            .client(OkHttpClient.Builder().build())
            .build()
            .create(EditsAPI::class.java)

        val editInsertResponse = editsRetrofit.postEditInsert("Bearer $token", "{패키지명}")



        println("edit insert response code: ${editInsertResponse.code()}")
        println("edit insert response body: ${editInsertResponse.body()}")

        exitProcess(0)
    }


}

fun getAccessToken(): String {
    val credentials =
        ServiceAccountCredentials.fromStream(FileInputStream("{Service Account Key 경로}"))
            .createScoped(setOf("https://www.googleapis.com/auth/androidpublisher"))
    credentials.refreshIfExpired()
    return credentials.accessToken.tokenValue
}

```


이렇게 edits를 생성할 수 있습니다.



## Edit Validate

insert로 생성한 edits가 만료되었는지 확인할 수 있는 API 입니다.

만료가 되었다면 response code가 400이 되고, 만료가 되지 않았다면 200이 됨과 동시에 Insert와 똑같은 형식의 Json 파일을 반환합니다.

[Google Play Developer API 문서](https://developers.google.com/android-publisher/api-ref/rest/v3/edits/validate)를 참고하여 아래와 같은 코드를 EditsAPI.kt에 추가합니다.


**EditsAPI.kt**
```kotlin

import retrofit2.Response
import retrofit2.http.Header
import retrofit2.http.POST
import retrofit2.http.Path

interface EditsAPI {

    @POST("{packageName}/edits")
    suspend fun postInsertEdit(
        @Header("Authorization")
        token: String,
        @Path("packageName")
        packageName: String,
    ): Response<EditResponse>

    //아래의 메소드를 추가
    @POST("{packageName}/edits/{editId}:validate")
    suspend fun postValidateEdit(
        @Header("Authorization")
        token: String,
        @Path("packageName")
        packageName: String,
        @Path("editId")
        editId:String
    ): Response<EditResponse>

}
```

그 다음 Validate를 사용하기 위해 아래와 같은 코드를 Main.kt에 추가합니다.


**Main.kt**

```kotlin
import com.google.auth.oauth2.ServiceAccountCredentials
import com.jakewharton.retrofit2.adapter.kotlin.coroutines.CoroutineCallAdapterFactory
import kotlinx.coroutines.*
import okhttp3.OkHttpClient
import retrofit2.Retrofit
import retrofit2.converter.gson.GsonConverterFactory
import java.io.FileInputStream
import java.util.*
import kotlin.system.exitProcess

fun main(args: Array<String>) {

    runBlocking {
        val token = getAccessToken()
        val editsRetrofit = Retrofit.Builder()
            .baseUrl("https://androidpublisher.googleapis.com/androidpublisher/v3/applications/")
            .addConverterFactory(GsonConverterFactory.create())
            .addCallAdapterFactory(CoroutineCallAdapterFactory())
            .client(OkHttpClient.Builder().build())
            .build()
            .create(EditsAPI::class.java)

        val editInsertResponse = editsRetrofit.postEditInsert("Bearer $token", "{패키지명}")

        println("edit insert response code: ${editInsertResponse.code()}")
        println("edit insert response body: ${editInsertResponse.body()}")

        val editInsertBody= editInsertResponse.body() ?: exitProcess(0)




        val editValidateResponse=editsRetrofit.postEditValidate("Bearer $token","{패키지명}",editInsertBody.id)

        println("edit validate response code: ${editValidateResponse.code()}")
        println("edit validate response body: ${editValidateResponse.body()}")

        exitProcess(0)
    }


}

fun getAccessToken(): String {
    val credentials =
        ServiceAccountCredentials.fromStream(FileInputStream("{Service Account Key 경로}"))
            .createScoped(setOf("https://www.googleapis.com/auth/androidpublisher"))
    credentials.refreshIfExpired()
    return credentials.accessToken.tokenValue
}

```
edit validate response code: 200이 출력되면 edit id의 edit이 유효하며 사용할 수 있는 상태 입니다.


## Edit delete

생성한 edit을 지우는 api 입니다.

edit id를 통하여 생성된 edit id를 지웁니다.


[Google Play Developer API 문서](https://developers.google.com/android-publisher/api-ref/rest/v3/edits/delete)를 참고하여 아래와 같은 코드를 EditsAPI.kt에 작성합니다.

**EditsAPI.kt**

```kotlin
import retrofit2.Response
import retrofit2.http.DELETE
import retrofit2.http.Header
import retrofit2.http.POST
import retrofit2.http.Path

interface EditsAPI {

    @POST("{packageName}/edits")
    suspend fun postInsertEdit(
        @Header("Authorization")
        token: String,
        @Path("packageName")
        packageName: String,
    ): Response<EditResponse>

    @POST("{packageName}/edits/{editId}:validate")
    suspend fun postValidateEdit(
        @Header("Authorization")
        token: String,
        @Path("packageName")
        packageName: String,
        @Path("editId")
        editId:String
    ): Response<EditResponse>

    //아래의 코드를 추가
    @DELETE("{packageName}/edits/{editId}")
    suspend fun deleteDeleteEdit(
        @Header("Authorization")
        token: String,
        @Path("packageName")
        packageName: String,
        @Path("editId")
        editId: String
    ): Response<Unit>

}

```

그 다음 Main.kt를 아래와 같이 수정합니다.

**Main.kt**

```kotlin
import com.google.auth.oauth2.ServiceAccountCredentials
import com.jakewharton.retrofit2.adapter.kotlin.coroutines.CoroutineCallAdapterFactory
import kotlinx.coroutines.*
import okhttp3.OkHttpClient
import retrofit2.Retrofit
import retrofit2.converter.gson.GsonConverterFactory
import java.io.FileInputStream
import java.util.*
import kotlin.system.exitProcess

fun main(args: Array<String>) {

    runBlocking {
        val token = getAccessToken()
        val editsRetrofit = Retrofit.Builder()
            .baseUrl("https://androidpublisher.googleapis.com/androidpublisher/v3/applications/")
            .addConverterFactory(GsonConverterFactory.create())
            .addCallAdapterFactory(CoroutineCallAdapterFactory())
            .client(OkHttpClient.Builder().build())
            .build()
            .create(EditsAPI::class.java)

        val editInsertResponse = editsRetrofit.postEditInsert("Bearer $token", "{패키지명}")

        println("edit insert response code: ${editInsertResponse.code()}")
        println("edit insert response body: ${editInsertResponse.body()}")

        val editInsertBody= editInsertResponse.body() ?: exitProcess(0)




        val editValidateResponse=editsRetrofit.postEditValidate("Bearer $token","{패키지명}",editInsertBody.id)

        println("edit validate response code: ${editValidateResponse.code()}")
        println("edit validate response body: ${editValidateResponse.body()}")

        val editDeleteResponse=editsRetrofit.deleteEditDelete("Bearer $token", "{패키지명}",editInsertBody.id)

        println("edit delete response code: ${editDeleteResponse.code()}")
        println("edit delete response body: ${editDeleteResponse.body()}")

        val editValidateResponse2=editsRetrofit.postEditValidate("Bearer $token","{패키지명}",editInsertBody.id)

        println("edit validate 2 response code: ${editValidateResponse2.code()}")
        println("edit validate 2 response body: ${editValidateResponse2.body()}")

        exitProcess(0)
    }


}

fun getAccessToken(): String {
    val credentials =
        ServiceAccountCredentials.fromStream(FileInputStream("{Service Account Key 경로}"))
            .createScoped(setOf("https://www.googleapis.com/auth/androidpublisher"))
    credentials.refreshIfExpired()
    return credentials.accessToken.tokenValue
}

```
edit delete response code: 204, edit validate 2 response code: 400가 출력되면 성공입니다.

## Edit commit

commit은 edit의 변경사항을 제출하는 API입니다.

이 API가 호출되면 변경사항이 Google Play에 제출되어 [Google Play Console](https://play.google.com/console/about/)의 앱이 검토중으로 상태가 바뀌게 됩니다.

따라서 이후에 작성할 Edit을 수정하는 API를 먼저 확인 후, 변경하길 원하는 요소를 변경한 후 호출하시기 바랍니다.

이번에는 사용 방법만 설명하겠습니다.

먼저 [Google Play Developer API 문서](https://developers.google.com/android-publisher/api-ref/rest/v3/edits/commit)를 참고하여 아래와 같은 코드를 EditsAPI.kt에 추가합니다.


```kotlin
import retrofit2.Response
import retrofit2.http.DELETE
import retrofit2.http.Header
import retrofit2.http.POST
import retrofit2.http.Path

interface EditsAPI {

    @POST("{packageName}/edits")
    suspend fun postInsertEdit(
        @Header("Authorization")
        token: String,
        @Path("packageName")
        packageName: String,
    ): Response<EditResponse>

    @POST("{packageName}/edits/{editId}:validate")
    suspend fun postValidateEdit(
        @Header("Authorization")
        token: String,
        @Path("packageName")
        packageName: String,
        @Path("editId")
        editId:String
    ): Response<EditResponse>

    @DELETE("{packageName}/edits/{editId}")
    suspend fun deleteDeleteEdit(
        @Header("Authorization")
        token: String,
        @Path("packageName")
        packageName: String,
        @Path("editId")
        editId: String
    ):Response<Unit>

    //아래의 코드를 추가

    @POST("{packageName}/edits/{editId}:commit")
    suspend fun postCommitEdit(
        @Header("Authorization")
        token: String,
        @Path("packageName")
        packageName: String,
        @Path("editId")
        editId: String,
    ):Response<EditResponse>

}
```

그 다음 아래의 코드룰 Main.kt의 **중간**에 추가합니다.

**Main.kt**

```kotlin
import com.google.auth.oauth2.ServiceAccountCredentials
import com.jakewharton.retrofit2.adapter.kotlin.coroutines.CoroutineCallAdapterFactory
import kotlinx.coroutines.*
import okhttp3.OkHttpClient
import retrofit2.Retrofit
import retrofit2.converter.gson.GsonConverterFactory
import java.io.FileInputStream
import kotlin.system.exitProcess

fun main(args: Array<String>) {

    runBlocking {
        val token = getAccessToken()
        val editsRetrofit = Retrofit.Builder()
            .baseUrl("https://androidpublisher.googleapis.com/androidpublisher/v3/applications/")
            .addConverterFactory(GsonConverterFactory.create())
            .addCallAdapterFactory(CoroutineCallAdapterFactory())
            .client(OkHttpClient.Builder().build())
            .build()
            .create(EditsAPI::class.java)

        val editInsertResponse = editsRetrofit.postInsertEdit("Bearer $token", "{패키지명}")

        println("edit insert response code: ${editInsertResponse.code()}")
        println("edit insert response body: ${editInsertResponse.body()}")

        val editInsertBody= editInsertResponse.body() ?: exitProcess(0)




        val editValidateResponse=editsRetrofit.postValidateEdit("Bearer $token","{패키지명}",editInsertBody.id)

        println("edit validate response code: ${editValidateResponse.code()}")
        println("edit validate response body: ${editValidateResponse.body()}")
        
        
        //이곳에 추가
        
        val editCommitResponse=editsRetrofit.postCommitEdit("Bearer $token","{패키지명}",editInsertBody.id)
        
        println("edit commit response code: ${editCommitResponse.code()}")
        println("edit commit response body: ${editCommitResponse.body()}")
        
        
        
        

        val editDeleteResponse=editsRetrofit.deleteDeleteEdit("Bearer $token", "{패키지명}",editInsertBody.id)

        println("edit delete response code: ${editDeleteResponse.code()}")
        println("edit delete response body: ${editDeleteResponse.body()}")

        val editValidateResponse2=editsRetrofit.postValidateEdit("Bearer $token","{패키지명}",editInsertBody.id)

        println("edit validate 2 response code: ${editValidateResponse2.code()}")
        println("edit validate 2 response body: ${editValidateResponse2.body()}")

        exitProcess(0)
    }


}

fun getAccessToken(): String {
    val credentials =
        ServiceAccountCredentials.fromStream(FileInputStream("{Service Account Key 경로}"))
            .createScoped(setOf("https://www.googleapis.com/auth/androidpublisher"))
    credentials.refreshIfExpired()
    return credentials.accessToken.tokenValue
}

```


이것으로 Edit을 생성, 유효성 검사, 삭제 제출하는 방법을 배웠습니다.

다음은 editId를 이용하여 앱의 정보를 수정하는 방법을 알아보겠습니다.









