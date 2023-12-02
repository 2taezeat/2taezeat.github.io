---
layout: post
title: "Kotlin Flow (2), Flow 개요"
date: 2023-11-12 01:54:46+0900
author: 2taezeat
toc: true
categories:

- blog
- Kotlin
- Flow

---
**Flow란 무엇인가?**
<!--more-->
---

# 플로우란 무엇인가?

- 플로우(flow)는 비동기적으로 계산해야 할 값의 스트림을 나타낸다.
- 플로우는 시퀀스와 달리 코루틴을 지원하며, 비동기적으로 계산되는 값을 나타낸다.
- Flow 인터페이스 자체는 떠다니는 원소들을 모으는 역할을 하며, 플로우의 끝에 도달할 때까지 각 값을 처리하는 것 의미한다.
    - Flow의 collect 는 컬렉션의 forEach와 비슷하다.
    - Flow의 유일한 멤버 함수는 collect 이다. 다른 함수는 확장 함수로 정의되어 있다.

```kotlin
interface Flow<out T> {
    suspend fun collect(collector: FlowCollector<T>)
}
```

- 시퀀스의 최종 연산은 중단 함수가 아니기 때문에, 시퀀스 빌더 내부에 중단점이 있다면 값을 기다리는 스레드가 **블로킹** 된다.
    - 따라서 sequencee 빌더의 스코프에서는 yield, yieldAll 외에 다른 **중단 함수**를 사용할 수 없다.
    - Sequence의 iterator가 중단 함수가 아니기 때문에, 시퀀스의 원소를 소비할 때 **블로킹이 되는 것이 문제가 된다.**

```kotlin
// Don't do that, we should use Flow instead of Sequence
fun allUsersSequence(
    api: UserApi
): Sequence<User> = sequence {
        var page = 0
        do {
            val users = api.takePage(page++)
            //suspend 함수 임으로, so compilation error
            yieldAll(users)
        } while (!users.isNullOrEmpty())
    }

```

```kotlin
// 하나의 코루틴이 다른 코루틴을 블로킹하게 된다.
// 같은 스레드에서 launch로 시작된 코루틴이 대기 하게 된다.
fun getSequence(): Sequence<String> = sequence {
        repeat(3) {
            Thread.sleep(1000)
            // the same result as if there were delay(1000) here
            yield("User$it")
        }
    }

suspend fun main() {
    withContext(newSingleThreadContext("main")) {
        launch {
            repeat(3) {
                delay(100)
                println("Processing on coroutine")
            }
        }

        val list = getSequence()
        list.forEach { println(it) }
    }
}
// (1 sec)
// User0
// (1 sec)
// User1
// (1 sec)
// User2
// Processing on coroutine
// (0.1 sec)
// Processing on coroutine
// (0.1 sec)
// Processing on coroutine
```

- Sequence를 사용했기 때문에 forEach가 블로킹 연산이 된다.
    - 따라서 같은 스레드에서 launch로 시작된 코루틴이 대기하게 되며, 하나의 코루틴이 다른 코루틴을 블로킹하게 된다.

- 이런 상황에서 Sequence 대신 **Flow**를 사용해야 한다.
- 플로우를 사용하면 코루틴이 연산을 수행하는 데 필요한 기능을 전부 사용할 수 있다.
- 플로우의 빌더와 연산은 `중단 함수(suspend)` 이며, `구조화된 동시성`과 적절한 예외 처리를 지원한다.

```kotlin
fun getFlow(): Flow<String> = flow {
    repeat(3) {
        delay(1000)
        emit("User$it")
    }
}

suspend fun main() {
    withContext(newSingleThreadContext("main")) {
        launch {
            repeat(3) {
                delay(100)
                println("Processing on coroutine")
            }
        }

        val list = getFlow()
        list.collect { println(it) }
    }
}
// (0.1 sec)
// Processing on coroutine
// (0.1 sec)
// Processing on coroutine
// (0.1 sec)
// Processing on coroutine
// (1 - 3 * 0.1 = 0.7 sec)
// User0
// (1 sec)
// User1
// (1 sec)
// User2
```

## 플로우의 특징

- **collect**(중단 함수)와 같은 플로우의 최종 연산은 **스레드를 블로킹**하는 대신 **코루틴을 중단**시킨다.
- 플로우는 `Coroutine Context` 를 활용하고, 예외를 처리하는 등의 **코루틴 기능**도 제공한다.
- 플로우 처리는 **취소** 가능하며, `구조화된 동시성`을 갖추고 있다.
- **flow 빌더는** 중단 함수가 아니며, 어떠한 스코프도 필요하지 않다.
- 플로우의 `최종 연산`은 **중단 가능**하며, 연산이 실행될 때, 부모 코루틴과의 관계가 정립된다. (`coroutineScope 함수`와 비슷)
    - `launch` 를 취소하면, 플로우 처리도 적절하게 취소된다.

## 플로우 명명법

- 플로우는 어딘가에서 시작 되어야 한다. (**플로우 빌더**, 다른 객체에서의 **변환** 또는 **헬퍼 함수**로 시작된다.)
- 플로우의 마지막 연산은 `최종 연산`이라고 불리며, **중단 가능**하거나 **스코프를 필요로** 하는 **유일한 연산**이다.
- `최종 연산`은 주로 람다 표현식을 가지거나 가지지 않는 **collect** 가 된다.
- 시작 연산과 최종 연산 사이에 플로우를 변경하는 `중간 연산`(intermediate operation을 가질 수 있다.

## 플로우 사용 예시

- 주로 **이벤트를 감지**해야 할 필요가 있을 때, 사용
- 서버가 보낸 이벤트를 통해 전달된 메시지를 받는 경우
- 텍스트 입력 또는 클릭과 같은 사용자 액션이 감지된 경우
- 센서 또는 위치나 지도와 같은 기기의 정보 변경을 받는 경우
- 데이터베이스의 변경을 감지하는 경우

플로우는 이 밖의 경우에도 **동시성 처리** 를 위해 유용하게 사용 될 수 있다.

```kotlin
suspend fun getOffers(
    sellers: List<Seller>
): List<Offer> = coroutineScope {
    sellers
        .map { seller ->
            async { api.requestOffers(seller.id) }
        }
        .flatMap { it.await() }
}
```

컬렉션 처리 내부에서 `async`를 사용하면 동시 처리를 할 수 있지만,
많은 요청을 한번에 보내면 client 뿐 아니라 server 모두에게 좋지 않다.

```kotlin
suspend fun getOffers(
    sellers: List<Seller>
): List<Offer> = sellers
    .asFlow()
    .flatMapMerge(concurrency = 20) { seller ->
        suspend { api.requestOffers(seller.id) }.asFlow()
    }
    .toList()
```

컬렉션 대신 플로우로 처리하면 동시 처리, 컨텍스트, 예외를 조절할 수 있다.

# Reference

- https://kotlinlang.org/docs/coroutines-guide.html
- 코틀린 코루틴 Kotlin Coroutines: Deep Dive (Marcin Moskała, 인사이트)