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
**Flow의 실제 구현**
<!--more-->
---

# 플로우의 실제 구현

- **기본적으로 플로우는 어떤 연산을 실행할지 정의한 것**
    - 중단 가능한 람다식에 몇 가지 요소를 추가하였다.

```kotlin
fun interface MyFlowCollector {
    suspend fun myEmit(value: String)
}

interface MyFlow {
    suspend fun myCollect(collector: MyFlowCollector)
}

fun myFlowBuilder(builder: suspend MyFlowCollector.() -> Unit) = object : MyFlow {
    override suspend fun myCollect(collector: MyFlowCollector) {
        collector.builder()
    }
}

suspend fun main() {
    val f: MyFlow = myFlowBuilder {
        myEmit("A")
        myEmit("B")
        myEmit("C")
    }
    f.myCollect { println(it) }
    f.myCollect { println(it) }
}
```

# Reference

- https://kotlinlang.org/docs/coroutines-guide.html
- 코틀린 코루틴 Kotlin Coroutines: Deep Dive (Marcin Moskała, 인사이트)