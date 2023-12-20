---
layout: post
title: "Android 개발시 Timber을 적용한 이유"
date: 2023-11-02 01:54:46+0900
author: 2taezeat
toc: true
categories:

- blog
- Android

---
**Android 개발시 Timber을 적용한 이유**
<!--more-->
---

# 개요

- Android 개발시 logging 에는 `Logcat` 을 기본으로 사용한다.
  - [https://developer.android.com/studio/debug/am-logcat?hl=ko](https://developer.android.com/studio/debug/am-logcat?hl=ko)

- `println` 으로도 logging 을 할 수 있지만, 단순 logcat 창에 출력 되는 기능만 가능하다.

- `logger` 라는 library 도 있지만, 2018.03 월 release 이후, update가 되지 않고 있다.
  - [https://github.com/orhanobut/logger](https://github.com/orhanobut/logger)
  - `logger` 는 더 pretty 하게 log 출력이 가능하지만, `timber`에 비해 좀 더 무겁다.

- `timber` 라는 libarary 를 사용하면, 커스텀도 가능하고, `Logcat`에 작성해야 하는 "TAG" 도 굳이 작성하지 않아도 자동으로 어느 파일에서 log가 남는지 알 수 있다.
  - `OkHttp`의 `HttpLoggingInterceptor` 에도 **timber**를 적용과 Custom이 가능하다.
  - 그 외에 logging 하는 방법은 기존 `Logcat` 과 동일하다. `Log.d( , )` -> `Timber.d()`

<img width="452" alt="스크린샷 2023-12-10 13 14 31" src="https://github.com/2taezeat/imageRepo/assets/59787852/3b59be5a-04be-47ed-b9ba-178da5c62a06">
<img width="598" alt="스크린샷 2023-12-10 13 14 19" src="https://github.com/2taezeat/imageRepo/assets/59787852/58f200a8-d717-44c4-a0b3-6c2f5a4e97a0">

# 세팅

- `gradle KTS` 을 사용한다고 가정
- `Version Catalog` 를 사용한다고 가정

`libs.versions.toml`

```toml
[versions]
timber = "5.0.1" #23.12.10 기준 최신 버전

[libraries]
timber = { module = "com.jakewharton.timber:timber", version.ref = "timber" }
```

app 모듈의 `build.gradle.kts`

```kotlin
dependencies {
    //...
    implementation(libs.timber)
}
```

`Application()`을 상속 받는 .kt 파일에 timber library을 initialize 한다.

```kotlin
class MainApplication : Application() {
    override fun onCreate() {
        super.onCreate()
        if (BuildConfig.DEBUG) {
            Timber.plant(Timber.DebugTree())
        }
    }
}
```

`AndroidManifest` 파일

```manifest
    <application
        android:name=".MainApplication"
        android:icon="@mipmap/ic_launcher"
        //...
```

# Custom logging with timber

- `createStackElementTag` 함수를 **override** 해서, logging 되는 **line, method, class**를 logcat에 자동으로 표시할 수 있게 된다.

```kotlin
class MainApplication : Application() {
    override fun onCreate() {
        super.onCreate()
        if (BuildConfig.DEBUG) {
            Timber.plant(object : Timber.DebugTree() {
                override fun createStackElementTag(element: StackTraceElement): String? {
                    return String.format(
                        "Class:%s: Line: %s, Method: %s",
                        super.createStackElementTag(element),
                        element.lineNumber,
                        element.methodName
                    )
                }
            })
        } else {
            Timber.plant(ReleaseTree())
        }
    }
}
```

`ReleaseTree` 를 따로 생성하여, **release** 모드 일때, **error report** 를 보낼 수 있게 할 수 있다.

```kotlin
class ReleaseTree : @NotNull Timber.Tree() {
    override fun log(priority: Int, tag: String?, message: String, t: Throwable?) {
        if (priority == Log.ERROR || priority == Log.WARN) {
            //SEND ERROR REPORTS TO YOUR Crashlytics.
        }
    }
}
```

# 장점

- **No need to write TAGs**
  - Timber가 자동으로 TAGs 를 생성한다.

- **Customized behavior**
  - Custom Timber 트리를 사용하여 Logcat에 로깅하는 대신, Crashlytics service로 log를 전송할 수 있다.

- **Customized Meta-Data**
  - `createStackElementTag` 함수를 통해 line, method, class를 logcat에 자동으로 표시할 수 있게 된다.

- **Lightweight**
  - 이미 존재하는 log utility 을 wrapper 한 library 여서 app 용량에 부담을 주지 않는다.
  - 참고 : timber version 5.0.0 부터 timber 코드가 java 가 아닌 kotlin 으로 변경되었다.
    - [https://github.com/JakeWharton/timber/blob/trunk/CHANGELOG.md](https://github.com/JakeWharton/timber/blob/trunk/CHANGELOG.md)

- **주기적으로 업데이트 되는 open-source**
  - 2021.08.13 에 5.0.1 버전이 release 되었다.
  - 다른 logging 라이브러리(ex.`logger`)에 비해 비교적 최근 까지 업데이트 된다.

# 참고 사항

- `HttpLoggingInterceptor` 사용시, `log` 함수를 **override** 하여, **debug** 모드 일때만, logging 하게 할 수 있다.
  - Custom 하여, Json을 pretty 하게 출력할 수 도 있다.

```kotlin
@Singleton
@Provides
fun provideLoggingInterceptor(): HttpLoggingInterceptor { // Hilt로 DI를 하는 경우
    val logger = HttpLoggingInterceptor.Logger { message -> Timber.tag("okHttp").d(message) }
    return HttpLoggingInterceptor(logger)
        .setLevel(HttpLoggingInterceptor.Level.BODY)
}
```

- `Timber.d("")` 처럼 `""`(_empty string_) 을 logging 하려면, Custom을 해야 한다.

```kotlin
override fun d(message: String?, vararg args: Any?) {
    if (message?.length == 0) {
        super.d("EMPTY_STRING", *args)
    } else {
        super.d(message, *args)
    }
}

```

# Reference

- [https://github.com/JakeWharton/timber](https://github.com/JakeWharton/timber)
- [https://medium.com/free-code-camp/how-to-log-more-efficiently-with-timber-a3f41b193940](https://medium.com/free-code-camp/how-to-log-more-efficiently-with-timber-a3f41b193940)
- [관련 PR Url](https://github.com/boostcampwm2023/and04-catchy-tape/pull/93)