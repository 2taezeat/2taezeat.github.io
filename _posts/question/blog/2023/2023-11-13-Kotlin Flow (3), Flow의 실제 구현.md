---
layout: post
title: "Kotlin Flow (3), Flow의 실제 구현"
date: 2023-11-13 01:54:46+0900
author: 2taezeat
toc: true
categories:

- blog
- Kotlin
- Flow

---
**Flow의 실제 내부 구현은 어떻게 되어 있을까?**
<!--more-->
---

# 플로우의 실제 구현

- **기본적으로 플로우는 어떤 연산을 실행할지 정의한 것**
    - 중단 가능한 람다식에 몇 가지 요소를 추가하였다.

```kotlin
interface MyFlow {
    suspend fun myCollect(collector: MyFlowCollector)
}

fun myFlowBuilder(actionBlock: suspend MyFlowCollector.() -> Unit) = object : MyFlow {
    override suspend fun myCollect(collector: MyFlowCollector) {
        collector.actionBlock()
    }
}

fun interface MyFlowCollector { // SAM 사용 
    suspend fun myEmit(value: String)
}

suspend fun main() { // SAM 사용 
    val f: MyFlow = myFlowBuilder {
        this.myEmit("A") // this = myFlowCollector, 생략 가능
        myEmit("B") 
    }
    f.myCollect { println(it) }
}

///////////////////////////////////////////////////////
interface MyFlowCollector { // SAM 미 사용
    suspend fun myEmit(value: String)
}

suspend fun main() { // SAM 미 사용
    val f: MyFlow = myFlowBuilder {
        this.myEmit("A") // this = myFlowCollector, 생략 가능
        myEmit("B")
    }
    f.myCollect(object : MyFlowCollector {
        override suspend fun myEmit(value: String) {
            println(value)
        }
    })
}

```

- **collect**를 호출하면, **flow 빌더**를 호출 할 때 넣은 **람다식**이 실행된다.
- 빌더의 람다식이 **emit을 호출**하면, **collect**가 호출되었을 때 **명시된 람다식**이 호출된다.

## Flow 처리 방식

- 플로우의 각 원소를 변환하는 map 함수
    - `map 함수`는 새로운 플로우를 만들기 때문에, **flow 빌더**를 사용한다.
    - 플로우가 시작되면, 래핑하고 있는 플로우를 시작하게 되므로, 빌더 내부에서 **collect** 메소드를 호출한다.
    - 원소를 **받을 때마다**, map은 원소를 변환하고 **새로운 플로우로 방출**한다.

```kotlin
fun <T, R> Flow<T>.map(
    transformation: suspend (T) -> R
): Flow<R> = flow {
    collect {
        emit(transformation(it))
    }
}

suspend fun main() {
    flowOf("A", "B", "C")
        .map {
            delay(1000)
            it.lowercase()
        }
        .collect { println(it) }
}
// (1 sec)
// a
// (1 sec)
// b
// (1 sec)
// c
```

## 동기로 작동하는 Flow

- 플로우 또한 `중단 함수` 처럼 **동기**로 작동한다.
    - 플로우가 완료될 때까지 **collect** 호출이 **중단(suspend)** 된다.
    - 즉, 플로우는 새로운 코루틴을 시작하지 않는다.
    - `중단 함수`가 코루틴을 시작할 수 있는 것처럼, 플로우의 각 단계에서도 코루틴을 시작할 수 있지만 중단 함수의 기본 동작은 아니다.
- 플로우의 각각의 처리 단계는 **동기**로 실행되기 때문에, onEach 내부에 **delay**가 있으면 모든 원소가 처리되기 전이 아닌 원소 사이에 지연이 생긴다.

```kotlin
suspend fun main() {
    flowOf("A", "B", "C")
        .onEach { delay(1000) }
        .collect { println(it) }
}
// (1 sec)
// A
// (1 sec)
// B
// (1 sec)
// C
```

## 플로우와 공유 상태

- 커스텀한 *플로우 처리 함수를 구현*할 때, 플로우의 각 단계가 **동기**로 작동하기 때문에 **동기화** 없이도 플로우 내부에 변경 가능한 상태를 정의할 수 있다.

- 플로우 단계 외부의 변수를 추출해서, 함수에서 사용하는 것이 흔히 저지르는 실수 중 하나이다.
    - 외부 변수는 같은 플로우가 모으는, 모든 코루틴이 공유하게 된다.
    - 이런 경우 **동기화**가 필수이며, **플로우 컬렉션이** 아니라 **플로우**에 종속되게 된다.
    - 따라서 두 개의 코루틴이 병렬로 원소를 세게 된다.

```kotlin
fun Flow<*>.counter() = flow<Int> {
    var counter = 0
    collect {
        counter++
        // to make it busy for a while
        List(100) { Random.nextLong() }.shuffled().sorted()
        emit(counter)
    }
}

suspend fun main(): Unit = coroutineScope {
    val f1 = List(1000) { "$it" }.asFlow()
    val f2 = List(1000) { "$it" }.asFlow().counter()

    launch { println(f1.counter().last()) } // 1000
    launch { println(f1.counter().last()) } // 1000
    launch { println(f2.last()) } // 1000
    launch { println(f2.last()) } // 1000
}
```

```kotlin
fun Flow<*>.counter(): Flow<Int> {
    var counter = 0
    return this.map {
        counter++
        List(100) { Random.nextLong() }.shuffled().sorted()
        counter
    }
}

suspend fun main(): Unit = coroutineScope {
    val f1 = List(1000) { "$it" }.asFlow()
    val f2 = List(1000) { "$it" }.asFlow().counter()

    launch { println(f1.counter().last()) } // 1000
    launch { println(f1.counter().last()) } // 1000
    launch { println(f2.last()) } // less than 2000
    launch { println(f2.last()) } // less than 2000
    // 콘솔 결과 창에서, 4개 print 결과 중 2개는 1000, 나머지는 2개는 2000보다 작다.
}
```

같은 변수를 사용하는 중단 함수들에서 동기화가 필요한 것처럼,

플로우에서 사용하는 변수가 **함수 외부, 클래스의 스코프, 최상위 레벨**에서 정의되어 있으면 **동기화**가 필요하다.

```kotlin
var counter = 0 // 최상위 레벨 변수

fun Flow<*>.counter(): Flow<Int> = this.map {
    counter++
    List(100) { Random.nextLong() }.shuffled().sorted()
    counter
}

suspend fun main(): Unit = coroutineScope {
    val f1 = List(1000) { "$it" }.asFlow()
    val f2 = List(1000) { "$it" }.asFlow().counter()

    launch { println(f1.counter().last()) } // 3496, less than 4000
    launch { println(f1.counter().last()) } // 3735
    launch { println(f2.last()) } // 3870
    launch { println(f2.last()) } // 3933
}
```

# Reference

- https://kotlinlang.org/docs/coroutines-guide.html
- 코틀린 코루틴 Kotlin Coroutines: Deep Dive (Marcin Moskała, 인사이트)