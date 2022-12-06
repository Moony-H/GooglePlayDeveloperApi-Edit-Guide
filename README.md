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
        ServiceAccountCredentials.fromStream(FileInputStream("/Users/hanmunhwi/Desktop/Google Play Console Key/pc-api-5791105689854140514-65-6f334fc49580.json"))


}

```

그 다음 현재 사용할 api의 권한 scope가 필요합니다. 따라서 아래와 같이 scope를 추가하고 refresh 합니다.

```kotlin
import com.google.auth.oauth2.ServiceAccountCredentials
import java.io.FileInputStream

fun main(args: Array<String>) {
    val credentials =
        ServiceAccountCredentials.fromStream(FileInputStream("/Users/hanmunhwi/Desktop/Google Play Console Key/pc-api-5791105689854140514-65-6f334fc49580.json"))
            .createScoped(setOf("https://www.googleapis.com/auth/androidpublisher")) //edit을 사용하기 위한 권한 scope.
    credentials.refreshIfExpired()

}
```

이제 access token을 활용하여 edit에 접근할 수 있습니다.

## Edit 생성

이제 insert를 사용하여 edit을 생성하겠습니다.


