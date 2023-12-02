---
layout: post
title: "Kotlin Flow (1), Hot vs Cold"
date: 2023-11-11 01:54:46+0900
author: 2taezeat
toc: true
categories:

- blog
- Kotlin
- Flow

---
**Hot vs Cold 데이터 스트림에 대해 알아 본다.**
<!--more-->
---



# Hot stream, Cold stream

- Hot : 컬렉션(List, Set) / Channel
- Cold : Flow, Sequence, RxJava 스트림


- Hot 데이터 스트림은 열정적이라 데이터를 소비하는 것과 무관하게 원소를 생성
    - Hot 데이터 스트림의 빌더와 연산은 **즉각 실행**된다.
    - 항상 사용 가능한 상태
    - 각 연산이 `최종 연산`이 될 수 있다.
    - 여러 번 사용됬을 때 매번 결과를 다시 계산할 필요가 없다.
    - 가능한 빨리 원소를 만들고 저장하며, 원소가 소비되는 것과 무관하게 생성한다.

- Cold 데이터 스트림은 게을러서 요청이 있을 때만 작업을 수행하며, 아무것도 저장하지 않습니다.
    - **원소가 필요할 때 까지 연산이 실행되지 않는다.**
    - 무한할 수 있다.
    - 원소를 어떻게 계산할지 **정의**할 것에 불과
    - 최소한의 연산만 수행
    - 중간에 생성되는 값을 저장하지 않기 때문에, 메모리를 적게 사용
    - **`최종 연산`이 모든 작업을 실행**
    - `최종 연산`에서 값이 필요할 때가 되어서야 처리한다.
    - **`중간 연산` 이전에 만든 시퀀스에 새로운 연산을 첨가할 뿐**
    - 중간 과정의 모든 함수는 무엇ㅇ르 해야 할지만 정의한 것.
  


```kotlin
fun m(i: Int): Int {
    print("m$i ")
    return i * i
}

fun f(i: Int): Boolean {
    print("f$i ")
    return i >= 10
}

fun main() {
    listOf(1, 2, 3, 4, 5, 6, 7, 8, 9, 10)
        .map { m(it) }
        .find { f(it) }
        .let { print(it) }
    // m1 m2 m3 m4 m5 m6 m7 m8 m9 m10 f1 f4 f9 f16 16

    sequenceOf(1, 2, 3, 4, 5, 6, 7, 8, 9, 10)
        .map { m(it) }
        .find { f(it) }
        .let { print(it) }
    // m1 f1 m2 f4 m3 f9 m4 f16 16
}
```


## Hot Channel, Cold Flow

- Channel은 핫 이라 값을 곧바로 계산, 소비되는 것과 상관없이 값을 생성한 뒤에 가지게 된다.
    - 수신자가 얼마나 많은지에 대해선 신경쓰지 않는다.
    - 각 원소는 단 한번만 받을 수 있기 때문에, 첫 번째 수신자가 모든 원소를 소비하고 나면 두 번째 소비자는 채널이 비어 있으며, 어떤 원소도 받을 수 없다.


- Flow는 Cold 데이터 소스 이기 때문에 값이 필요할 때만 생성, `flow 빌더` 자체는 어떤 연산을 하지 않는다.
    - `flow` 는 단지 (*collect*와 같은) 최종 연산이 호출될 때 원소가 어떻게 생성되어야 하는지 **정의**한 것에 불과
    - 따라서, **flow 빌더**는 `CoroutineScope` 가 필요하지 않는다.
    - **flow 빌더**는 빌더를 호출한 `최종 연산`의 스코프에서 실행된다.
    - 플로우의 각 최종 연산은 처음부터 데이터를 처리하기 시작한다.


```kotlin
private fun CoroutineScope.makeChannel() = produce {
    println("Channel started")
    for (i in 1..3) {
        delay(1000)
        send(i)
    }
}

suspend fun main() = coroutineScope {
    val channel = makeChannel()

    delay(1000)
    println("Calling channel...")
    for (value in channel) {
        println(value)
    }
    println("Consuming again...")
    for (value in channel) {
        println(value)
    }
}
// Channel started
// (1 sec)
// Calling channel...
// 1
// (1 sec)
// 2
// (1 sec)
// 3
// Consuming again...
```

```kotlin
private fun makeFlow() = flow {
    println("Flow started")
    for (i in 1..3) {
        delay(1000)
        emit(i)
    }
}

suspend fun main() = coroutineScope {
    val flow = makeFlow()

    delay(1000)
    println("Calling flow...")
    flow.collect { value -> println(value) }
    println("Consuming again...")
    flow.collect { value -> println(value) }
}
// (1 sec)
// Calling flow...
// Flow started
// (1 sec)
// 1
// (1 sec)
// 2
// (1 sec)
// 3
// Consuming again...
// Flow started
// (1 sec)
// 1
// (1 sec)
// 2
// (1 sec)
// 3
```


# Reference

- https://kotlinlang.org/docs/coroutines-guide.html
- 코틀린 코루틴 Kotlin Coroutines: Deep Dive (Marcin Moskała, 인사이트)