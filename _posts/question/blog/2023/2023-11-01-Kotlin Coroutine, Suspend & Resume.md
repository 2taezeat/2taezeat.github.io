---
layout: post
title: "Kotlin Coroutine, Suspend & Resume"
date: 2023-11-01 01:54:46+0900
author: 2taezeat
toc: true
categories:

- blog
- Kotlin
- Coroutine

---
**Kotlin Coroutine Suspend, Resume을 정리 한다.**

<!--more-->
---


# Suspend (중단)

- 스레드는 저장이 불가능하고 멈추는 것만 가능하지만, 코루틴은 중단하고 재개가 가능하다

- 코루틴은 중단 되었을때, `Continuation` 객체를 반환한다.

- suspend 함수는 코루틴을 중단할 수 있는 함수
    - suspend 함수는 코루틴 또는 다른 suspend 함수에서 호출 되어야한다.
    - suspend 함수는 중단할 수 있는 중단점이 필요하다.

```kotlin
suspend fun main() {
    println("Before")
    println("After")
}
// Before
// After
```

```kotlin
suspend fun main() {
    println("Before")
    suspendCoroutine<Unit> { }
    println("After")
}
// Before
```

```kotlin
//3
suspend fun main() {
    println("Before")
    suspendCoroutine<Unit> { continuation ->
        println("Before too")
    } // 람다 표현식이 suspend 되기 전에 실행된다.
    println("After")
}
// Before
// Before too

```

# Resume (재개)

- 중단된 코루틴은 `Continuation` 객체를 이용해 재개 가능하다.
- 다른 스레드에서 코루틴을 재개할 수 있다.
- 코루틴에서는 값으로 재개 한다.(`Continuation`의 제네릭 타입 인자)

```kotlin
suspend fun main() {
    println("Before")
    suspendCoroutine<Unit> { continuation ->
        continuation.resume(Unit)
    }
    println("After")
}
// Before
// After
```

```kotlin
suspend fun main() {
    println("Before")

    suspendCoroutine<Unit> { continuation ->
        thread {
            println("Suspended")
            Thread.sleep(1000)
            continuation.resume(Unit)
            println("Resumed")
        }
    }
    println("After")
}
// Before
// Suspended
// (1 second delay)
// After
// Resumed

```

```kotlin
fun requestUser(callback: (User) -> Unit) {
    thread {
        Thread.sleep(1000)
        callback.invoke(User("Test"))
    }
}

suspend fun main() {
    println("Before")
    val user = suspendCoroutine<User> { cont ->
        requestUser { user -> cont.resume(user) }
    }
    println(user)
    println("After")
}
// Before
// (1 second delay)
// User(name=Test)
// After
```

suspend 와 resume 을 통해, 스레드는 다른 일을 할 수 있고, 값 또는 데이터가 도착하면 스레드는 코루틴이 중단된 지점에서 재개 하게 됩니다. 이때 resume 함수와 `Continuation` 객체를
통해 값을 얻을 수 있습니다.

## Continuation.kt (code 중 일부)

```kotlin
/**
 * Interface representing a continuation after a suspension point that returns a value of type `T`.
 */
public interface Continuation<in T> {
    /**
     * The context of the coroutine that corresponds to this continuation.
     */
    public val context: CoroutineContext

    /**
     * Resumes the execution of the corresponding coroutine passing a successful or failed [result] as the
     * return value of the last suspension point.
     */
    public fun resumeWith(result: Result<T>)
}

/**
 * Resumes the execution of the corresponding coroutine passing [value] as the return value of the last suspension point.
 */
public inline fun <T> Continuation<T>.resume(value: T): Unit =
    resumeWith(Result.success(value))

/**
 * Resumes the execution of the corresponding coroutine so that the [exception] is re-thrown right after the
 * last suspension point.
 */
public inline fun <T> Continuation<T>.resumeWithException(exception: Throwable): Unit =
    resumeWith(Result.failure(exception))


public suspend inline fun <T> suspendCoroutine(crossinline block: (Continuation<T>) -> Unit): T {
    contract { callsInPlace(block, InvocationKind.EXACTLY_ONCE) }
    return suspendCoroutineUninterceptedOrReturn { c: Continuation<T> ->
        val safe = SafeContinuation(c.intercepted())
        block(safe)
        safe.getOrThrow()
    }
}

```

# Reference

- https://kotlinlang.org/docs/coroutines-guide.html
- 코틀린 코루틴 Kotlin Coroutines: Deep Dive (Marcin Moskała, 인사이트)