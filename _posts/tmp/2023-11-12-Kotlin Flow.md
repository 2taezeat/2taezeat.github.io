- [플로우 만들기](#-------)
  * [원시값을 가지는 플로우](#------------)
  * [컨버터](#---)
  * [함수를 플로우로 바꾸기](#------------)
  * [플로우 빌더](#------)
  * [플로우 빌더 이해하기](#-----------)
- [플로우 생명주기 함수](#-----------)
  * [onEach](#oneach)
  * [onStart](#onstart)
  * [onCompletion](#oncompletion)
  * [onEmpty](#onempty)
  * [catch](#catch)
  * [잡히지 않은 예외](#---------)
  * [flowOn](#flowon)
  * [launchIn](#launchin)
- [플로우 처리](#------)
  * [map](#map)
  * [filter](#filter)
  * [take](#take)
  * [drop](#drop)
  * [컬렉션 처리는 어떻게 작동할까?](#-----------------)
  * [merge](#merge)
  * [zip](#zip)
  * [combine](#combine)
  * [fold와 scan](#fold--scan)
  * [flatMapConcat](#flatmapconcat)
  * [flatMapMerge](#flatmapmerge)

---

# 플로우 생명주기 함수

- 플로우는 요청이 한ㅉ고 방향으로 흐르고, 요청에 의해 생성된 값이 다른 방향으로 흐르는 파이프라 생각할 수 있다.
- 플로우가 완료되거나 예외가 발생했을 때, 이러한 저어보가 전달되어 중간 단계가 종료된다.

## onEach

- 플로우의 값을 하나씩 받기 위해 onEach 함수를 사용합니다.
- onEach 람다식은 중단 함수이며, 원소는 순서대로 처리 됩니다.

## onStart

- onStart 함수는 최종 연산이 호출될 때와 같이 플로우가 시작되는 경우에 호출되는 리스너를 설정한다.
- onStart 는 첫 번째 원소를 '요청' 했을 때 호출되는 합수다.
- onStart 에서도 원소를 내보낼 수 있습니다. 원소들은 onStart 부터 아래로 흐르게 된다.

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

- 플로우를 완료할 수 있는, 가장 흔한 방법
- 플로우 빌더가 끝났을 때(예를 들면, 마지막 원소가 전송되었을 때)
- onCompleteion 메서드를 사용해 플로우가 완료되었을 때 호출되는 리스너를 추가할 수 있습니다.
- 안드로이드 에서 네트워크 응답을 기다리고 있는 척도인 Progress Bar를 보여주기 위해 `onStart`를 사용하며, 가리기 위해서는 `onCompletion`을 사용한다.

## onEmpty

- 플로우는 예기치 않은 이벤트가 발생하면 값을 내보내기 전에 완료될 수 있다.
- onEmpty 함수는 원소를 내보내기 전에 플로우가 완료되면 실행됩니다.
- onEmpty 기본값을 내보내기 위한 목적으로 사용될 수 있습니다.

## catch

- 플로우를 만들거나 처리하는 도중에 예외가 발생할 수 있다.
- 이러한 예외는 아래로 흐르면서 처리하는 단계를 하나씩 닫는다.
- 하지만 예외를 잡고 관리 할 수도 있다.
- 리스너는 예외 인자로 받고 정리를 위한 연산을 수행할 수 있습니다.
- catch 메서드는 예외를 잡아 전파되는 걸 멈춘다. 이전 단계는 이미 완료된 상태지만, catch는 새로운 값을 여전히 내보낼 수 있어 남은 플로우를 지속할 수 있다.
- catch 하수는 윗부분에서 던진 예외에만 반응한다. (예외는 아래로 흐를 때 잡는다고 생각하면 된다.)

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

- 플로우에서 잡히지 않은 예외는 플로우를 즉시 취소하며, collect는 예외를 다시 던집니다.
    - 중단 함수, coroutineScope 모두 같은 방식으로 예외를 처리한다.
- catch를 사용하는 건(마지막 연산 뒤에 catch가 올 수 없기 때문에) 최종 연산에서 발생한 예외를 처리하는 데 전혀 도움이 되지 않는다.
    - 따라서 collect에서 예외가 발생하면 에외를 잡지 못하게 되어 블록 밖으로 예외가 전달된다.
- 그러므로, collect의 연산을 onEach로 옮기고 catch 이전에 두는 방법이 자주 사용된다.
    - collect의 연산을 옮긴다면 catch가 모든 예외를 잡을 거라고 확신할 수 있다.

```kotlin
//11
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
//12
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

- onEach, onStart, onCompletiom과 같은 플로우 연산과 flow나 channelFlow와 같은 플로우 빌더의 인자로 사용되는 람다식은 모두 둥단 함수이다.
- 중단 함수는 컨텍스트가 필요하며(구조화된 동시성을 위해) 부모와 관계를 유지합니다
- 플로우의 함수들은 collect가 호출되는 곳의 컨텍스트에서 컨텍스트를 얻는다.
- flowOn 함수로 컨텍스트를 변경할 수 있다.

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

- collect는 플로우가 완료될 때 까지 코루틴을 중단하는 중단 연산이다.
- launch 빌더로 collect를 래핑하면 플로우를 다른 코루틴에서 처리할 수 있다.
- 플로우의 확장 함수인 launchIn을 사용하면 유일한 인자로 스코프를 받아 collect를 새로운 코루틴에서 시작할 수 있다.
- 별도의 코루틴에서 플로우를 시작하기 위해 launchIn을 주로 사용한다.

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

# 플로우 처리

- 플로우 생성과 최종 연산 사이의 값을 변경하는 연산들을 플로우 처리(flow processing) 이라고 한다.

## map

- 플로우의 각 원소를 변환 함수에 따라 변환하는 map 함수.

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

- 원래 플로우에서 주어진 조건에 맞는 값들만 가진 플로우를 반환한다.
- 관심 없는 원소를 제거할 때 주로 사용

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

- 특정 수의 원소만 통과시킬 때 사용

## drop

- 특정 수의 원소를 무시할 때 사용

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

## 컬렉션 처리는 어떻게 작동할까?

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

## merge

- 두 개의 플로우를 하나의 플로우로 합칠 때 사용되는 함수 => merge, zip, combine
- merge: 두 개의 플로우에서 생성된 원소들을 하나로 합칠 때 사용
    - merge를 사용하면 한 플로우의 원소가 다른 플로우를 기다리지 않는다는 것이 중요하다.
    - 첫 번째 플로우의 원소 생성이 지연된다고 해서 두 번째 플로우의 원소 생성이 중단되지 않는다.

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

- 두 플로우로 부터 쌍을 만들때 사용
- 원소가 쌍을 이루는 방법을 정하는 함수도 필요하다.
- 각 원소는 한 쌍의 일부가 되므로 쌍이 될 원소를 기다려야 한다.
- 쌍을 이루지 못하고 남은 원소는 유실되므로 한 플로우에서 zipping이 완료되면 생성되는 플로우 또한 완료된다.
- zip은 쌍을 필요로 하기 때문에 첫 번째 플로우가 닫히면 함수 또한 끝나게 된다.

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

- combine을 사용하면 모든 새로운 원소가 전임자를 '대체' 하게 된다. (zip의 경우 느린 플로우를 기다려야 한다.)
- 첫 번째 쌍이 이미 만들어졌다면 다른 플로우의 이전 원소와 함께 새로운 쌍이 만들어진다.
- combine은 두 플로우 모두 닫힐 때까지 원소를 내보낸다.
- combine은 두 데이터 소스의 변화를 능동적으로 감지할 때 주로 사용된다.

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

- 변화가 발생할 때마다 원소가 내보내지길 원한다면 합쳐질 각 플로우에 초기 값을 더하면 된다.

```kotlin
userUpdateFlow.onStart { emit(currentUser) }
```

- View가 감지 가능한 원소 두 가지 중 하나라도 변경 될 때 반응해야 하는 경우 combine을 주로 사용

```kotlin
userStateFlow
    .combine(notificationsFlow) { userState, notifications ->
        updateNotificationBadge(userState, notifications)
    }
    .collect()
```

## fold와 scan

- 초기 값부터 시작하여 주어진 원소 각각에 대해 두 개의 값을 하나로 합치는 연산을 적용하여 컬렉션의 모든 값을 하나로 합칠때 사용
- fold는 최종 연산이고, 플로우에서도 사용할 수 있으며, collect 처럼 플로우가 완료될 때까지 중단(suspend) 된다.

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

- scan은 누적되는 과정의 모든 값을 생성하는 중간 연산이다.
- scan은 이전 단계에서 값을 받은 즉시 새로운 값을 만들기 때문에 Flow에서 유용하게 사용된다.

```kotlin
fun main() {
    val list = listOf(1, 2, 3, 4)
    val res = list.scan(0) { acc, i -> acc + i }
    println(res) // [0, 1, 3, 6, 10]
}
```

## flatMapConcat

- 컬렉션의 경우 flatMap은 map 과 비슷하지만 변환 함수가 '평탄화된 컬렉션' 을 반환해야 한다는 점이 다르다.
- 플로우의 경우 변환 함수가 평탄화된 플로우를 반환한다고 생각하는게 직관적이다.
    - 문제는 플로우 원소가 나오는 시간이 다르다는 점이다.
    - 이러한 이유로 Flow에는 flatMap 하수가 없으며, 대신 flatMapConcat, flatMapMerge, flatMapLatest와 같은 함수가 존재한다.
- flatMapConcat 함수는 생성된 플로우를 하나씩 처리한다.
    - 그래서 두 번째 플로우는 첫 번째 플로우가 완료되었을 때 시작할 수 있다.

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

- 만들어진 플로우를 동시에 처리한다.

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
- concurreny 인자를 사용해 동시에 처리할 수 있는 플로우의 수를 설정할 수 있다.
  - 기본값은 16이지만, 변경 가능하다.

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

- flatMapMerge는 플로우의 각 원소에 대한 데이터를 요청할 때 주로 사용된다.
  - 예를 들어 종류를 목록으로 가지고 있다면 종류별로 요청을 보내야 한다.
  - async 함수 대신, 플로우와 함께 flatMapMerge를 사용하면 두 가지 이점이 있다.
    - 동시성 인자를 제어하고(같은 시간에 수백 개의 요청을 보내는 걸 피하기 위해) 같은 시간에 얼마만큼의 종류를 처리할지 결정할 수 있다.
    - Flow를 반환하여 데이터가 생성될 때마다 다음 원소를 보낼 수 있다.(함수를 사용하는 측면에서 보면 데이터를 즉시 처리할 수 있다.)

```kotlin
suspend fun getOffers(
    categories: List<Category>
): List<Offer> = coroutineScope {
    categories
        .map { async { api.requestOffers(it) } }
        .flatMap { it.await() }
}

// A better solution
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
- 새로운 플로우가 나타나면 이전에 처리하던 플로우를 잊어버린다.

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

- 예외는 플로우를 따라 흐르면서 각 단계를 하나씩 종료한다.
- 종료된 단계는 비활성화되기 때문에, 예ㅚ가 발생한 뒤 메시지를 보내는 건 불가능하지만, 각 단계가 이전 단계에 대한 참조를 가지고 있으며, 플로우를 다시 시작하기 위해 참조를 사용할 수 있다.
- 이 원리에 기반하여, 코틀린은 retry와 retryWhen 함수를 제공한다.
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
    predicate: suspend (cause: Throwable) -> Boolean = {true}
): Flow<T> {
    require(retries > 0) {
      "Expected positive amount of retries, but had $retries"
    }
    return retryWhen { cause, attempt ->
        attempt < retries && predicate(cause)
    }
}
```
- retryWhen 은 플로우의 이전 단계에서 예외가 발생할 때마다 조건자(predicate) 를 확인한다.
- 몇 번까지 재시도할지와 특정 예외 클래스가 발생했을 때만 처리할지를 명시한다.

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

- 어떤 예외든지 항상 재시도 하는 경우, 로그를 남기고 새로운 연결 맺는 걸 시도할 때 시간 간격을 주기 위해 predict(조건자)를 정의한다.
```kotlin
fun makeConnection(config: ConnectionConfig) = api
    .startConnection(config)
    .retry { e ->
        delay(1000)
        log.error(e) { "Error for $config" }
        true
    }
```
- 연결을 계속해서 재시도할 때 시간 간격을 점진적으로 증가시키는 방법도 자주 사용된다.
- 예외가 특정 타입일 때 재시도하는 조건자를 구현할 수 도 있다.
```kotlin
fun makeConnection(config: ConnectionConfig) = api
    .startConnection(config)
    .retryWhen { e, attempt ->
        delay(100 * attempt)
        log.error(e) { "Error for $config" }
        e is ApiException && e.code !in 400..499
    }
```


## 중복 제거 함수 (distinctUntilChanged)
- 반복되는 원소가 동일하다고 판단되면 제거하는 distinctUntilChanged 함수도 아주 유용하다.
- distinctUntilChanged 함수는 '바로 이전의 원소'와 동일한 원소만 제거한다.
- distinctUntilChangedBy 는 두 원소가 동일한지 판단하기 위해 비교할 키 선택자를 인자로 받는다. (기본으로 사용되는 equals를 대신한다.)
```kotlin
fun <T> Flow<T>.distinctUntilChanged(): Flow<T> = flow {
        var previous: Any? = NOT_SET
        collect {
            if (previous == NOT_SET || previous != it) {
                emit(it)
                previous = it
            }
        }
    }

private val NOT_SET = Any()

suspend fun main() {
  flowOf(1, 2, 2, 3, 2, 1, 1, 3)
    .distinctUntilChanged()
    .collect { print(it) } // 123213
}
```

## 최종 연산
- 플로우를 처리를 끝내는 연산 => 최종 연산 이라고 부른다.
- collect 외에도, 컬렉션과 Sequence 가 제공하는 것과 비슷한
  - count, first, firstOrNull, fold, reduce 또한 최종 연산이다.
- 최종 연산은 중단 가능(suspend) 가능하며, 플로우가 완료되었을 때 또는 최종 연산 자체가 플로우를 완료 시켰을 때 값을 반환한다.
- collect 메서드를 사용해서 또 다른 최종 연산을 얼마든지 구현할 수 도 있다.
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