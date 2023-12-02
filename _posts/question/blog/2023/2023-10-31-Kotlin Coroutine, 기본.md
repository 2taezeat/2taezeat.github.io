---
layout: post
title: "Kotlin Coroutine, 기본"
date: 2023-10-31 01:54:46+0900
author: 2taezeat
toc: true
categories:

- blog
- Kotlin
- Coroutine

---
**Kotlin Coroutine 기본 개념을 정리 한다.**

<!--more-->
---

# 사전 정의

**비동기(Asynchronous)** : 작업의 요청과 그 결과가 순차적인 흐름을 따르지 않는 것
- 현재 작업의 응답이 끝나지 않은 상태에서 다음 작업이 요청된다.


**동시성(Concurrency)** : 여러 작업이 같은 시간에 실행 되어 보이는 상태
- 흔히 Single Core, Multi-Thread 환경에서 가능
- Context-Switch 발생
- 시분할(time-sharing)
- 동기화 이슈 발생
- 병행성 이라고도 불림

**병렬성(Parallelism)** : 여러 작업이 같은 시간에 실제로 여러 개 실행되는 상태
- 흔히 Multi Core, Multi-Thread 환경에서 가능
- 동기화 이슈 발생

**Thread Blocking** : 스레드가 일시 정지된 상태



# Kotlin Coroutine 기본
## 정의
> Coroutines are light-weight threads that allow you to write asynchronous non-blocking code.

> A coroutine is an instance of a suspendable computation.

=> suspend 와 resume이 가능한 프로그래밍 모듈
## 쓰임처
sequential code를 통해 asynchronous, non-blocking 프로그래밍을 할때 사용돤다.

```kotlin
fun showNews() {
  viewModelScope.launch {
      val config = getConfigFromApi()
      val news = getNewsFromApi(config)
      val user = getUserFromApi()
      view.showNews(user, news)
  }
}
```

## 대안
### callback
```kotlin
fun onCreate() {
    getNewsFromApi { 
        news -> val sortedNews = news.sorted()
        view.showNews(sortedNews)
    }
}
```
- callback 함수가 많아지면 가독성이 떨어진다. (callback 지옥)
- cancel (취소) 가 어렵다.
- callback 함수가 많아지면, 작업의 순서를 개발자가 다루기 힘들어진다.

### thread
```kotlin
fun onCreate() {
    thread {
        val news = getNewsFromApi()
        val sortedNews = news.sorted()
      
        runOnUiThread {
          view.showNews(sortedNews)
        }
    }
}
```
- 스레드가 실행된 이후 중단이 불가하다.
- 스레드는 생성 비용이 높다.
- 스레드를 자주 전환하면 관리하기 어렵고 복잡도가 증가한다.
- 가독성이 떨어진다.


### RxJava
```kotlin
fun onCreate() {
  disposables += getNewsFromApi()
      .subscribeOn(Schedulers.io())
      .observeOn(AndroidSchedulers.mainThread())
      .map { news ->
          news.sortedByDescending { it.publishedAt }
      }
      .subscribe { sortedNews ->
          view.showNews(sortedNews)
      }
}

```
- 러닝 커브가 높다.
- 구현하기 복잡하다.
  - end-point가 증가 할수록 더욱 복잡해진다.

## 특징
- 특정 thread에 종속되지 않는다
  - 하나의 Thread에서 suspend 되었다가 다른 Thread에서 resume 된다.
- suspend 시 스레드가 blocking 되지 않는다.
- 스레드에 비해 light-weighted(경량) 하다. 
- 비선점형 방식으로 동작
  - 실행 주체가 자신의 실행권을 자발적으로 내려 놓음 


# Reference
- https://kotlinlang.org/docs/coroutines-guide.html
- 코틀린 코루틴 Kotlin Coroutines: Deep Dive (Marcin Moskała, 인사이트)