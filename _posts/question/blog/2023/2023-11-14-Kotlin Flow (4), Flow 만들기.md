---
layout: post
title: "Kotlin Flow (4), Flow 만들기"
date: 2023-11-14 01:54:46+0900
author: 2taezeat
toc: true
categories:

- blog
- Kotlin
- Flow

---
**Flow 를 생성해보자.**
<!--more-->
---

# 플로우 만들기

## 원시값을 가지는 플로우

- 플로우를 만드는 가장 간단한 방법은 플로우가 어떤 값을 가져야 한느지 정의하는 `flowOf` 함수를 사용하는 것이다.
    - 리스트의 `listOf` 함수와 비슷하다.
- 값이 없는 플로우는 `emptyFlow()` 함수를 사용한다.

## 컨버터

- `asFlow` 함수를 사용해서 `Iterable, Iterator, Sequence`를 Flow로 바꿀 수 있다.
    - `asFlow` 함수는 즉시 사용 가능한 원소들의 플로우를 만든다.
    - 플로우 처리 함수를 사용해 처리 가능한 원소들의 플로우를 시작할때 유용하다.

```kotlin
suspend fun main() {
    listOf(1, 2, 3, 4, 5)
        // or setOf(1, 2, 3, 4, 5)
        // or sequenceOf(1, 2, 3, 4, 5)
        .asFlow()
        .collect { print(it) } // 12345
}
```

## 함수를 플로우로 바꾸기

- 플로우는 RxJava의 Single 처럼 **시간상 지연되는 하나의 값**을 나타낼 때 자주 사용된다.
    - 따라서 **중단 함수**를 **플로우로 변환**하는 것 또한 가능하다.
    - 이때 **중단 함수의 결과**가 **플로우의 유일한 값**이 된다.

```kotlin
suspend fun main() {
    val function = suspend {
        // this is suspending lambda expression
        delay(1000)
        "UserName"
    }

    function.asFlow()
        .collect { println(it) }
}
// (1 sec)
// UserName

suspend fun getUserName(): String {
    delay(1000)
    return "UserName"
}

suspend fun main() {
    ::getUserName
        .asFlow()
        .collect { println(it) }
}
// (1 sec)
// UserName
```

## 플로우 빌더

- 플로우를 만들 때 가장 많이 사용되는 방법인 `flow 빌더`는 **시퀀스**를 만드는 `sequence 빌더`나 **채널**을 만드는 `produce 빌더`와 비슷하게 작동한다.
    - 빌더는 `flow 함수`를 호출하고, 람다식 내부에서 **emit** 함수를 사용해 다음 값을 방출한다.
    - Channel이나 Flow에서 모든 값을 방출하려면 **emitAll** 를 사용할 수 있다.
        - `emitAll(flow)` 는 `flow.collect { emit(it) }`과 같다.

```kotlin
fun allUsersFlow(
    api: UserApi
): Flow<User> = flow {
    var page = 0
    do {
        val users = api.takePage(page++) // suspending
        emitAll(users)
    } while (!users.isNullOrEmpty())
}
```

## 플로우 빌더 이해하기

```kotlin
public fun <T> flowOf(vararg elements: T): Flow<T> = flow {
    for (element in elements) {
        emit(element)
    }
}
```

- `flow 빌더`는 간단하다. collect 메서드 내부에서 block 함수를 호출하는 `Flow 인터페이스`를 구현한다.

```kotlin
fun <T> flow(
    block: suspend FlowCollector<T>.() -> Unit
): Flow<T> = object : Flow<T>() {
    override suspend fun collect(collector: FlowCollector<T>) {
        collector.block()
    }
}

interface Flow<out T> {
    suspend fun collect(collector: FlowCollector<T>)
}

fun interface FlowCollector<in T> {
    suspend fun emit(value: T)
}

fun main() = runBlocking { // 예저
    flow { // 1
        emit("A")
        emit("B")
    }.collect { value -> println(value) // 2 }
}
// A
// B
```

- `flow 빌더`를 호출하면 단지 객체를 만들 뿐이다.
- 반면 **collect**를 호출하면 `collector 인터페이스`의 **block 함수**를 호출하게 된다. (예제 1에서 정의된 람다식)
- 리시버는 2에서 정의된 람다식인 **collect** 이다.
- `FlowCollector` 와 같이 `fun interface` 으로 정의하면, `람다식의 본체`는 `함수형 인터페이스의 함수 본체`로 사용된다. (여기서는 emit)
- 그러므로 emit 함수의 본체는 _println(value)_ 가 된다.
- 따라서, **collect를 호출**하면 1에서 정의된 람다식을 실행하기 시작하고, **emit을 호출** 했을 때 `정의된 람다식( println("A") )`을 호출한다.

# Reference

- https://kotlinlang.org/docs/coroutines-guide.html
- 코틀린 코루틴 Kotlin Coroutines: Deep Dive (Marcin Moskała, 인사이트)