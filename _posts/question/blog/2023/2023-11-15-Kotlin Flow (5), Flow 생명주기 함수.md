---
layout: post
title: "Kotlin Flow (5), Flow 생명주기 함수"
date: 2023-11-15 01:54:46+0900
author: 2taezeat
toc: true
categories:

- blog
- Kotlin
- Flow

---
**Flow 생명주기 함수를 알아보자.**
<!--more-->
---

# 플로우 생명주기 함수
- 플로우는 요청이 한쪽 방향으로 흐르고, 요청에 의해 생성된 값이 다른 방향으로 흐르는 파이프라 생각할 수 있다.
- 플로우가 **완료**되거나 **예외**가 발생했을 때, 이러한 정보가 전달되어 **중간 단계**가 **종료**된다.
- 모든 정보가 플로우로 전달되므로 **값, 예외, 시작, 완료** 같은 다른 이벤트를 감지할 수 있다.

## onEach
- 플로우의 값을 하나씩 받기 위해 `onEach` 함수를 사용한다.
- `onEach` 람다식은 **중단 함수**이며, 원소는 순서대로 처리 된다.

## onStart

- `onStart` 함수는 **최종 연산**이 호출될 때와 같이 플로우가 시작되는 경우에 호출되는 리스너를 설정한다.
- `onStart` 는 첫 번째 원소를 **'요청'** 했을 때 호출되는 함수이다.
    - `onStart`는 첫 번째 원소가 생성되는 걸 기다렸다 호출되는 게 아니다.
- `onStart `에서도 원소를 내보낼 수 있다.
    - 원소들은 `onStart` 부터 아래로 흐르게 된다.
  
```kotlin
suspend fun main() {
    flowOf(1, 2)
        .onEach { delay(1000) }
        .onStart { println("Before") }
        .collect { println(it) }
}
// Before
// (1 sec)
// 1
// (1 sec)
// 2
```

## onCompletion

- 플로우를 **완료**할 수 있는, 가장 흔한 방법이다.
    - **플로우 빌더**가 끝났을 때(예를 들면, 마지막 원소가 전송되었을 때)
- `onCompleteion` 메서드를 사용해 플로우가 완료되었을 때 호출되는 리스너를 추가할 수 있습니다.
- 안드로이드 에서 네트워크 응답을 기다리고 있는 척도인 Progress Bar를 보여주기 위해, `onStart`를 사용하며, 가리기 위해서는 `onCompletion`을 사용한다.

## onEmpty

- 플로우는 **예기치 않은 이벤트**가 발생하면 값을 내보내기 전에 **완료**될 수 있다.
- `onEmpty` 함수는 원소를 내보내기 전에 플로우가 **완료**되면 실행된다.
- `onEmpty` **기본 값**을 내보내기 위한 목적으로 사용될 수 있습니다.

## catch

- 플로우를 만들거나 처리하는 도중에 **예외**가 발생할 수 있다.
- 이러한 **예외**는 **아래로 흐르면서 처리하는 단계를 하나씩 닫는다.**
- 하지만 **예외**를 잡고 관리 할 수도 있다.
- 리스너는 **예외** 인자로 받고 정리를 위한 연산을 수행할 수 있습다.
- `catch` 메서드는 예외를 잡아 **전파되는 걸 멈춘다.**
    - 이전 단계는 이미 완료된 상태지만, `catch`는 새로운 값을 여전히 내보낼 수 있어 남은 플로우를 지속할 수 있다.
- `catch` 함수는 **윗부분에서 던진 예외에만 반응한다.**
    - **예외**는 아래로 흐를 때 잡는다고 생각하면 된다.

```kotlin
class MyError : Throwable("My error")

val flow = flow {
    emit("Message1")
    throw MyError()
}

suspend fun main(): Unit {
    flow.catch { emit("Error") }
        .collect { println("Collected $it") }
}
// Collected Message1
// Collected Error
```
## 잡히지 않은 예외

- **플로우에서 잡히지 않은 예외**는 플로우를 **즉시 취소**하며, `collect`는 **예외**를 다시 던집니다.
    - `중단 함수`, `coroutineScope` **모두 같은 방식**으로 **예외**를 처리한다.
- `catch`를 사용하는 건(마지막 연산 뒤에 `catch`가 올 수 없기 때문에) **최종 연산**에서 발생한 **예외**를 처리하는 데 전혀 도움이 되지 않는다.
    - 따라서 `collect`에서 **예외**가 발생하면 **에외**를 잡지 못하게 되어 블록 밖으로 **예외**가 전달된다.
- 그러므로, `collect`의 연산을 `onEach`로 옮기고 catch 이전에 두는 방법이 자주 사용된다.
    - collect의 연산을 옮긴다면 catch가 모든 예외를 잡을 거라고 확신할 수 있다.

```kotlin
class MyError : Throwable("My error")

val flow = flow {
    emit("Message1")
    emit("Message2")
}

suspend fun main(): Unit {
    flow.onStart { println("Before") }
        .catch { println("Caught $it") }
        .collect { throw MyError() }
}
// Before
// Exception in thread "..." MyError: My error
```

```kotlin
val flow = flow {
    emit("Message1")
    emit("Message2")
}

suspend fun main(): Unit {
    flow.onStart { println("Before") }
        .onEach { throw MyError() }
        .catch { println("Caught $it") }
        .collect()
}
// Before
// Caught MyError: My error
```

## flowOn

- `onEach, onStart, onCompletiom`과 같은 플로우 연산과 `flow나 channelFlow`와 같은 `플로우 빌더`의 **인자로 사용되는 람다식은 모두 둥단 함수**이다.
- **중단 함수**는 `context`가 필요하며(**구조화된 동시성**을 위해) 부모와 관계를 유지합니다
- 플로우의 함수들은 `collect`가 호출되는 곳의 `context`에서 `context`를 얻는다.
- `flowOn` 함수로 `context`를 변경할 수 있다.
- `flowOn`은 플로우에서 **윗부분에 있는 함수에서만 작동한다.**

```kotlin
suspend fun present(place: String, message: String) {
    val ctx = coroutineContext
    val name = ctx[CoroutineName]?.name
    println("[$name] $message on $place")
}

fun messagesFlow(): Flow<String> = flow {
    present("flow builder", "Message")
    emit("Message")
}

suspend fun main() {
    val users = messagesFlow()
    withContext(CoroutineName("Name1")) {
        users
            .flowOn(CoroutineName("Name3"))
            .onEach { present("onEach", it) }
            .flowOn(CoroutineName("Name2"))
            .collect { present("collect", it) }
    }
}
// [Name3] Message on flow builder
// [Name2] Message on onEach
// [Name1] Message on collect
```
## launchIn

- `collect`는 플로우가 **완료**될 때 까지 코루틴을 중단하는 **중단 연산**이다.
- `launch 빌더`로 `collect`를 래핑하면 **플로우를 다른 코루틴에서 처리할 수 있다.**
- 플로우의 확장 함수인 `launchIn`을 사용하면 유일한 인자로 **Scope**를 받아 collect를 **새로운 코루틴**에서 시작할 수 있다.
- **별도의 코루틴에서 플로우를 시작하기 위해** `launchIn`을 주로 사용한다.

```kotlin
fun <T> Flow<T>.launchIn(scope: CoroutineScope): Job =
    scope.launch { collect() }

suspend fun main(): Unit = coroutineScope {
    flowOf("User1", "User2")
        .onStart { println("Users:") }
        .onEach { println(it) }
        .launchIn(this)
}
// Users:
// User1
// User2
```

# Reference
- https://kotlinlang.org/docs/coroutines-guide.html
- 코틀린 코루틴 Kotlin Coroutines: Deep Dive (Marcin Moskała, 인사이트)