---
layout: post
title: "Kotlin Flow (6), Flow 처리(Processing)"
date: 2023-11-16 01:54:46+0900
author: 2taezeat
toc: true
categories:

- blog
- Kotlin
- Flow

---
**Flow 처리(Processing)에 대해 알아보자.**
<!--more-->
---
# 플로우 처리
- 플로우 **생성**과 **최종 연산** 사이의 값을 변경하는 연산들을 **플로우 처리(flow processing)** 이라고 한다.
- 플로우는 값이 흐르기 때문에 제외하고, 곱하고, 변형하거나, 합치는 등의 여러 가지 방법으로 변경 가능하다.

## 컬렉션 처리는 어떻게 작동할까?
- `flow 빌더`와 람다식을 가진 `collect` 만으로 구현 가능하다.

```kotlin
suspend fun main() {
    flowOf('a', 'b')
        .map { it.uppercase() }
        .collect { print(it) } // AB
}

fun <T, R> Flow<T>.map(
    transform: suspend (value: T) -> R
): Flow<R> = flow {
    collect { value -> emit(transform(value)) }
}

fun <T> flowOf(vararg elements: T): Flow<T> = flow {
    for (element in elements) {
        emit(element)
    }
}

suspend fun main() {
    flow map@{ // 1
        flow flowOf@{ // 2
            for (element in arrayOf('a', 'b')) { // 3
                this@flowOf.emit(element) // 4
            }
        }.collect { value -> // 5
            this@map.emit(value.uppercase()) // 6
        }
    }.collect { // 7
        print(it) // 8
    }
}
```

- **1**에서 플로우 시작, **7**에서 원소들을 모은다.
- 모으기 시작할 때, **1**에서 시작하는 `@map` 람다식을 수행하며, 이 람다식은 **2**에서 또 다른 빌더를 호출하고, **5**에서 원소들을 모은다.
- 원소 들을 모을 때, **2**에서 시작하는 `@flowOf` 람다식을 시작한다.
- **2**의 람다식은 'a', 'b'를 가진 배열을 탐색한다.
- 첫 번째 값인 'a'를 **4**에서 내보내며, **5** 람다식이 호출된다.
- **5**의 람다식은 갑을 'A'로 변경하며 `@map` 플로우로 내보낸 뒤, **7**의 람다식이 호출 된다.
- 값이 출력된 후 **7**의 람다식이 종료되고 **6**에서 람다식이 재개 된다.
- 람다식이 끝났기 때문에 **4**의 `@flowOf`가 재개되며, 탐색이 다시 시작되어 **4**에서 'b'를 내보낸다.
- **5**에서 람다식이 호출되고, 'B'로 값을 변형한 뒤 **6**에서 `@map` 플로우로 내보낸다.
- 값은 **7**에서 모이며 **8**에서 출력된다.
- **7**의 람다식이 종료되므로 **6**의 람다식이 재개 된다. 이 람다식도 종료되었기 때문에 **4**의 `@flowOf` 람다식이 다시 시작된다.
- **4**도 종료되었기 때문에 **5**의 `collect`에서 `@map`이 재개된다. 더 이상 남은 것이 없기 때문에 `@map`의 마지막 부분에 도달한다.
- **7**의 `collect`에서 다시 시작하면 `main 함수`의 끝에 도달한다.

## map
- 플로우의 각 원소를 **변환 함수**에 따라 변환하는 `map` 함수.
```kotlin
fun <T, R> Flow<T>.map(
    transform: suspend (value: T) -> R
): Flow<R> = flow { // here we create a new flow
    collect { value -> // here we collect from receiver
        emit(transform(value))
    }
}
```

## filter

- 원래 플로우에서 주어진 **조건에 맞는 값**들만 가진 플로우를 반환한다.
- 관심 없는 원소를 **제거**할 때 주로 사용

```kotlin
suspend fun main() {
    (1..10).asFlow() // [1, 2, 3, 4, 5, 6, 7, 8, 9, 10]
        .filter { it <= 5 } // [1, 2, 3, 4, 5]
        .filter { isEven(it) } // [2, 4]
        .collect { print(it) } // 24
}

fun isEven(num: Int): Boolean = num % 2 == 0
```
## take
- 특정 수의 원소만 **통과**시킬 때 사용

## drop
- 특정 수의 원소를 **무시**할 때 사용

```kotlin
suspend fun main() {
    ('A'..'Z').asFlow()
        .take(5) // [A, B, C, D, E]
        .collect { print(it) } // ABCDE
}

suspend fun main() {
    ('A'..'Z').asFlow()
        .drop(20) // [U, V, W, X, Y, Z]
        .collect { print(it) } // UVWXYZ
}
```

## merge

- 두 개의 플로우를 하나의 플로우로 **합칠 때** 사용되는 함수 => `merge, zip, combine`
- `merge`: 두 개의 플로우에서 생성된 원소들을 하나로 합칠 때 사용
    - `merge`를 사용하면 한 플로우의 원소가 다른 플로우를 **기다리지 않는다**는 것이 중요하다.
    - 첫 번째 플로우의 원소 생성이 **지연**된다고 해서, 두 번째 플로우의 원소 생성이 **중단되지 않는다.**
- **여러 개의 이벤트들을 똑같은 방법**으로 처리할 때 `merge`를 사용한다.

```kotlin
suspend fun main() {
    val ints: Flow<Int> = flowOf(1, 2, 3)
    val doubles: Flow<Double> = flowOf(0.1, 0.2, 0.3)

    val together: Flow<Number> = merge(ints, doubles)
    print(together.toList())
    // [1, 0.1, 0.2, 0.3, 2, 3]
    // or [1, 0.1, 0.2, 0.3, 2, 3]
    // or [0.1, 1, 2, 3, 0.2, 0.3]
    // or any other combination
}
```

```kotlin
suspend fun main() {
    val ints: Flow<Int> = flowOf(1, 2, 3)
        .onEach { delay(1000) }
    val doubles: Flow<Double> = flowOf(0.1, 0.2, 0.3)

    val together: Flow<Number> = merge(ints, doubles)
    together.collect { println(it) }
}
// 0.1
// 0.2
// 0.3
// (1 sec)
// 1
// (1 sec)
// 2
// (1 sec)
// 3
```

## zip

- 두 플로우로 부터 **쌍**을 만들때 사용한다.
- **원소가 쌍을 이루는 방법을 정하는 함수**도 필요하다.
- 각 원소는 한 쌍의 일부가 되므로 **쌍이 될 원소를 기다려야 한다.**
- 쌍을 이루지 못하고 남은 원소는 **유실**되므로 한 플로우에서 **zipping**이 완료되면 **생성되는 플로우 또한 완료된다.**
- `zip`은 쌍을 필요로 하기 때문에 **첫 번째 플로우가 닫히면 함수 또한 끝나게 된다.**

```kotlin
suspend fun main() {
    val flow1 = flowOf("A", "B", "C")
        .onEach { delay(400) }
    val flow2 = flowOf(1, 2, 3, 4)
        .onEach { delay(1000) }
    flow1.zip(flow2) { f1, f2 -> "${f1}_${f2}" }
        .collect { println(it) }
}
// (1 sec)
// A_1
// (1 sec)
// B_2
// (1 sec)
// C_3
```

## combine

- `combine`을 사용하면 모든 새로운 원소가 전임자를 **대체** 하게 된다.
    - `zip`의 경우 느린 플로우를 기다려야 한다.
- 첫 번째 쌍이 이미 만들어졌다면 다른 플로우의 이전 원소와 함께 **새로운 쌍이 만들어진다.**
- `combine`은 두 플로우 **모두 닫힐 때까지 원소를 내보낸다.**
- `combine`은 **두 데이터 소스의 변화를 능동적으로 감지할 때** 주로 사용된다.

```kotlin
suspend fun main() {
    val flow1 = flowOf("A", "B", "C")
        .onEach { delay(400) }
    val flow2 = flowOf(1, 2, 3, 4)
        .onEach { delay(1000) }
    flow1.combine(flow2) { f1, f2 -> "${f1}_${f2}" }
        .collect { println(it) }
}
// (1 sec)
// B_1
// (0.2 sec)
// C_1
// (0.8 sec)
// C_2
// (1 sec)
// C_3
// (1 sec)
// C_4

```

- 변화가 발생할 때마다 원소가 내보내지길 원한다면 합쳐질 각 플로우에 **초기 값을 더하면 된다.**
- Ex. `View`가 감지 가능한 원소 **두 가지 중에 하나라도 변경될 때 반응**해야 하는 경우 `combine`을 주로 사용한다.

```kotlin
userUpdateFlow.onStart { emit(currentUser) }

AFlow.combine(BFlow) { a, b ->
    updateView(a, b)
}.collect
```

## fold
- 초기 값부터 시작하여 주어진 원소 각각에 대해 **두 개의 값을 하나로 합치는 연산**을 적용하여 `컬렉션`의 모든 값을 하나로 합칠때 사용
- `fold`는 **최종 연산**이고, 플로우에서도 사용할 수 있으며, `collect` 처럼 플로우가 완료될 때까지 **중단(suspend)** 된다.

```kotlin
suspend fun main() {
    val list = flowOf(1, 2, 3, 4)
        .onEach { delay(1000) }
    val res = list.fold(0) { acc, i -> acc + i }
    println(res)
}
// (4 sec)
// 10
```
## scan
- `scan`은 **누적되는 과정의 모든 값을 생성**하는 **중간 연산**이다.
- `scan`은 이전 단계에서 값을 받은 **즉시 새로운 값을 만들기** 만든다.

## flatMapConcat

- `컬렉션`의 경우 `flatMap`은 `map` 과 비슷하지만 변환 함수가 **평탄화된 컬렉션** 을 반환해야 한다는 점이 다르다.
- 플로우의 경우 변환 함수가 **평탄화된 플로우**를 반환한다고 생각하는게 직관적이다.
    - 문제는 플로우 원소가 나오는 **시간이 다르다는 점이다.**
    - 이러한 이유로 플로우에는 `flatMap` 함수가 없으며, 대신 `flatMapConcat, flatMapMerge, flatMapLatest`와 같은 함수가 존재한다.
- `flatMapConcat` 함수는 생성된 플로우를 **하나씩 처리한다.**
    - 그래서 두 번째 플로우는 첫 번째 플로우가 **완료**되었을 때 시작할 수 있다.

```kotlin
fun flowFrom(elem: String) = flowOf(1, 2, 3)
    .onEach { delay(1000) }
    .map { "${it}_${elem} " }

suspend fun main() {
    flowOf("A", "B", "C")
        .flatMapConcat { flowFrom(it) }
        .collect { println(it) }
}
// (1 sec)
// 1_A
// (1 sec)
// 2_A
// (1 sec)
// 3_A
// (1 sec)
// 1_B
// (1 sec)
// 2_B
// (1 sec)
// 3_B
// (1 sec)
// 1_C
// (1 sec)
// 2_C
// (1 sec)
// 3_C
```

## flatMapMerge

- 특징: 만들어진 플로우를 **동시에** 처리한다.

```kotlin
fun flowFrom(elem: String) = flowOf(1, 2, 3)
    .onEach { delay(1000) }
    .map { "${it}_${elem} " }

suspend fun main() {
    flowOf("A", "B", "C")
        .flatMapMerge { flowFrom(it) }
        .collect { println(it) }
}
// (1 sec)
// 1_A
// 1_B
// 1_C
// (1 sec)
// 2_A
// 2_B
// 2_C
// (1 sec)
// 3_A
// 3_B
// 3_C
```
- `concurreny` 인자를 사용해 **동시에** 처리할 수 있는 플로우의 수를 설정할 수 있다.
    - 기본값은 **16**이지만, 변경 가능하다.

```kotlin
fun flowFrom(elem: String) = flowOf(1, 2, 3)
    .onEach { delay(1000) }
    .map { "${it}_${elem} " }

suspend fun main() {
    flowOf("A", "B", "C")
        .flatMapMerge(concurrency = 2) { flowFrom(it) }
        .collect { println(it) }
}
// (1 sec)
// 1_A
// 1_B
// (1 sec)
// 2_A
// 2_B
// (1 sec)
// 3_A
// 3_B
// (1 sec)
// 1_C
// (1 sec)
// 2_C
// (1 sec)
// 3_C
```

- `flatMapMerge`는 플로우의 **각 원소에 대한 데이터를 요청**할 때 주로 사용된다.
- 예를 들어 종류를 목록으로 가지고 있다면, 종류별로 요청을 보내야 한다.
- `async` 함수 대신, 플로우와 함께 `flatMapMerge`를 사용하면 두 가지 이점이 있다.
    - **동시성 인자**를 제어하고(같은 시간에 수백 개의 요청을 보내는 걸 피하기 위해) 같은 시간에 얼마만큼의 종류를 처리할지 **결정**할 수 있다.
    - `Flow`를 반환하여 데이터가 생성될 때마다, 다음 원소를 보낼 수 있다.
        - 함수를 사용하는 측면에서 보면, 데이터를 즉시 처리할 수 있다.

```kotlin
suspend fun getOffers(
    categories: List<Category>
): List<Offer> = coroutineScope {
    categories
        .map { async { api.requestOffers(it) } }
        .flatMap { it.await() }
}

// 더 나은 방법 이다.
suspend fun getOffers(
    categories: List<Category>
): Flow<Offer> = categories
    .asFlow()
    .flatMapMerge(concurrency = 20) {
        suspend { api.requestOffers(it) }.asFlow()
        // or flow { emit(api.requestOffers(it)) }
    }

```

## flatMapLatest
- 특징: 새로운 플로우가 나타나면 이전에 처리하던 플로우를 **잊어버린다.**

```kotlin
fun flowFrom(elem: String) = flowOf(1, 2, 3)
    .onEach { delay(1000) }
    .map { "${it}_${elem} " }

suspend fun main() {
    flowOf("A", "B", "C")
        .flatMapLatest { flowFrom(it) }
        .collect { println(it) }
}
// (1 sec)
// 1_C
// (1 sec)
// 2_C
// (1 sec)
// 3_C
```

```kotlin
fun flowFrom(elem: String) = flowOf(1, 2, 3)
    .onEach { delay(1000) }
    .map { "${it}_${elem} " }

suspend fun main() {
    flowOf("A", "B", "C")
        .onEach { delay(1200) }
        .flatMapLatest { flowFrom(it) }
        .collect { println(it) }
}
// (2.2 sec)
// 1_A
// (1.2 sec)
// 1_B
// (1.2 sec)
// 1_C
// (1 sec)
// 2_C
// (1 sec)
// 3_C
```

## retry(재시도)

- **예외**는 플로우를 따라 흐르면서 **각 단계를 하나씩 종료한다.**
- 종료된 단계는 **비활성화**되기 때문에, **예외**가 발생한 뒤 메시지를 보내는 건 불가능하지만, **각 단계**가 **이전 단계에 대한 참조**를 가지고 있으며, 플로우를 **다시 시작**하기 위해 **참조**를 사용할 수 있다.
- 이 원리에 기반하여, 코틀린은 `retry`와 `retryWhen` 함수를 제공한다.
```kotlin
fun <T> Flow<T>.retryWhen(
    predicate: suspend FlowCollector<T>.(
        cause: Throwable,
        attempt: Long,
    ) -> Boolean,
): Flow<T> = flow {
    var attempt = 0L
    do {
        val shallRetry = try {
            collect { emit(it) }
            false
        } catch (e: Throwable) {
            predicate(e, attempt++)
                .also { if (!it) throw e }
        }
    } while (shallRetry)
}
```

```kotlin
fun <T> Flow<T>.retry(
    retries: Long = Long.MAX_VALUE,
    predicate: suspend (cause: Throwable) -> Boolean = { true }
): Flow<T> {
    require(retries > 0) {
        "Expected positive amount of retries, but had $retries"
    }
    return retryWhen { cause, attempt ->
        attempt < retries && predicate(cause)
    }
}
```
- `retryWhen` 은 플로우의 이전 단계에서 **예외**가 발생할 때마다 `조건자(predicate)` 를 확인한다.
- 몇 번까지 **재시도**할지와 **특정 예외 클래스**가 발생했을 때만 처리할지를 명시한다.

```kotlin
suspend fun main() {
    flow {
        emit(1)
        emit(2)
        error("E")
        emit(3)
    }.retry(3) {
        print(it.message)
        true
    }.collect { print(it) } // 12E12E12E12(exception thrown)
}
```

- 어떤 **예외**든지 항상 **재시도** 하는 경우, **log**를 남기고 새로운 연결 맺는 걸 시도할 때 **시간 간격**을 주기 위해 **predict(조건자)** 를 정의한다.
```kotlin
fun makeConnection(config: ConnectionConfig) = api
    .startConnection(config)
    .retry { e ->
        delay(1000)
        log.error(e) { "Error for $config" }
        true
    }
```
- 연결을 계속해서 **재시도**할 때 **시간 간격**을 점진적으로 증가시키는 방법도 자주 사용된다.
- **예외가 특정 타입**일 때 **재시도**하는 **조건자**를 구현할 수 도 있다.
```kotlin
fun makeConnection(config: ConnectionConfig) = api
    .startConnection(config)
    .retryWhen { e, attempt ->
        delay(100 * attempt)
        log.error(e) { "Error for $config" }
        e is ApiException && e.code !in 400..499
    }
```

## 최종 연산
- 플로우를 처리를 끝내는 연산 => **최종 연산** 이라고 부른다.
- `collect` 외에도, `Collection(컬렉션)`과 `Sequence` 가 제공하는 것과 비슷한 `count, first, firstOrNull, fold, reduce` 또한 **최종 연산**이다.
- **최종 연산**은 **중단 가능(suspend)** 가능하며, 플로우가 **완료**되었을 때 또는 **최종 연산 자체가 플로우를 완료** 시켰을 때 **값을 반환**한다.
- `collect` 메서드를 사용해서 또 다른 **최종 연산**을 얼마든지 **구현**할 수 도 있다.
```kotlin
suspend fun main() {
    val flow = flowOf(1, 2, 3, 4) // [1, 2, 3, 4]
        .map { it * it } // [1, 4, 9, 16]

    println(flow.first()) // 1
    println(flow.count()) // 4

    println(flow.reduce { acc, value -> acc * value }) // 576
    println(flow.fold(0) { acc, value -> acc + value }) // 30
}
```
# Reference
- https://kotlinlang.org/docs/coroutines-guide.html
- 코틀린 코루틴 Kotlin Coroutines: Deep Dive (Marcin Moskała, 인사이트)