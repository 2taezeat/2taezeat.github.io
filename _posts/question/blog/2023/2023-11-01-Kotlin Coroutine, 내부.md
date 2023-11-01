---
layout: post
title: "Kotlin Coroutine, 내부"
date: 2023-11-01 02:54:46+0900
author: 2taezeat
toc: true
categories:

- blog
- Kotlin
- Coroutine

---
Kotlin Coroutine 내부 동작을 정리 한다.

예상 독자: Kotlin Coroutine 을 사용해보고, 비동기 프로그래밍을 해본 사람
<!--more-->

# 

코틀린 중단 함수는 `continuation-passing style(CPS)` 로 구현되어 있다.

```kotlin
suspend fun getUser(): User?
suspend fun setUser(user: User)
suspend fun checkAvailability(flight: Flight): Boolean

// 내부
fun getUser(continuation: Continuation<*>): Any?
fun setUser(user: User, continuation: Continuation<*>): Any
fun checkAvailability(flight: Flight, continuation: Continuation<*>): Any
```

`Continuation` 객체는 상태를 나타내는 숫자와 로컬 데이터를 가지고 있다.

`Continuation` 은 함수 인자로 전달된다.

중단 함수는 상태(state) 를 가지는 state machine 으로 볼 수 있다.

중단 함수의 `Continuation` 객체가 이 함수를 부르는 다른 함수의 `Continuation` 객체를 decorate(장식) 한다.

모든 `Continuation` 객체는 resume 하거나 함수가 완료 될때 사용되는 콜 스택으로 사용된다.

현재 상태를 저장에는 label 이라는 field 를 사용한다. (처음에는 0으로 설정)
이후에는 중단되기 전에 다음 상태로 설정되어 재개 될 시점을 알 수 있게 해준다.

```kotlin
suspend fun myFunction() {
    println("Before")
    var counter = 0
    delay(1000) // suspending
    counter++
    println("Counter: $counter")
    println("After")
}

```

```kotlin
// 내부를 kotlin 으로 변환
fun myFunction(continuation: Continuation<Unit>): Any {
    val continuation = continuation as? MyFunctionContinuation
        ?: MyFunctionContinuation(continuation)

    var counter = continuation.counter

    if (continuation.label == 0) {
        println("Before")
        counter = 0
        continuation.counter = counter
        continuation.label = 1
        if (delay(1000, continuation) == COROUTINE_SUSPENDED) {
            return COROUTINE_SUSPENDED
        }
    }
    if (continuation.label == 1) {
        counter = (counter as Int) + 1
        println("Counter: $counter")
        println("After")
        return Unit
    }
    error("Impossible")
}

class MyFunctionContinuation(
    val completion: Continuation<Unit>
) : Continuation<Unit> {
    override val context: CoroutineContext
        get() = completion.context

    var result: Result<Unit>? = null
    var label = 0
    var counter = 0

    override fun resumeWith(result: Result<Unit>) {
        this.result = result
        val res = try {
            val r = myFunction(this)
            if (r == COROUTINE_SUSPENDED) return
            Result.success(r as Unit)
        } catch (e: Throwable) {
            Result.failure(e)
        }
        completion.resumeWith(res)
    }
}
```

# 콜 스택과 Continuation 객체
함수 a가 함수 b를 호출하면 가상 머신은 a의 상태와 b가 끝나면 실행이 될 지점을 콜 스택(call stack) 에 저장한다.

코루틴을 중단하면 스레드를 반환해 콜 스택에 있는 저오가 사라진다.

따라서 코루틴을 재개 할 때, 콜 스택을 사용할 수 없다.

대신 `Continuation` 객체가 콜 스택의 역할을 대신한다.

`Continuation` 객체는 중단 되었을때 상태(label)와 함수의 지역 변수와 파라미터(필드), 중단 함수를 호출한 함수가 재개될 위치 정보를 가지고 있다.

하나의 `Continuation` 객체는 다른 하나를 참조하는 식으로 동작한다.

`Continuation` 객체가 재개 될 때, 각 `Continuation` 객체는 자신이 담당하는 함수를 호출하고, 그 함수의 실행이 끝나면, 자신을 호출한 함수의 `Continuation` 객체을 재개 합니다. 
이 과정은 스택의 끝에 다다를 때까지 반복된다.

중단된 함수가 재개했을 때 `Continuation` 객체로 부터 상태를 복원하고, 얻은 결과를 사용하거나 예외를 던진다.

# Reference

- https://kotlinlang.org/docs/coroutines-guide.html
- 코틀린 코루틴 Kotlin Coroutines: Deep Dive (Marcin Moskała, 인사이트)