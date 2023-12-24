---
layout: post
title: "Kotlin Flow (7), SharedFlow, StateFlow"
date: 2023-11-17 01:54:46+0900
author: 2taezeat
toc: true
categories:

- blog
- Kotlin
- Flow

---
**SharedFlow, StateFlow 에 대해 알아보자.**
<!--more-->
---
# SharedFlow

- SharedFlow를 통해 메시지를 보내면, 대기하고 있는 모든 코루틴이 수신하게 된다.
    - 브로드캐스트 채널과 비슷하다.

```kotlin
suspend fun main(): Unit = coroutineScope {
    val mutableSharedFlow =
        MutableSharedFlow<String>(replay = 0)
    // or MutableSharedFlow<String>()

    launch {
        mutableSharedFlow.collect { println("#1 received $it") }
    }
    launch {
        mutableSharedFlow.collect { println("#2 received $it") }
    }

    delay(1000)
    mutableSharedFlow.emit("Message1")
    mutableSharedFlow.emit("Message2")
}
// (1 sec)
// #1 received Message1
// #2 received Message1
// #1 received Message2
// #2 received Message2
// (program never ends)
```
> 위 프로그램은 coroutineScope의 자식 코루틴이 launch로 시작된 뒤 MutableSharedFlow를 감지하고 있는 상태 임으로 종료되지 않는다.
> 프로그램을 종료하려면 전체 스코프를 취소해야 한다.

- MutabledSharedFlow는 메시지 보내는 작업을 유지 할 수도 있다.
- 기본값이 0인 replay 인자를 설정하면 마지막으로 전송한 값들이 정해진 수 만큼 저장된다.
- 코루틴이 감지를 시작하면 저장된 값들을 먼저 받게 된다.
- resetReplayCache를 사용하면 값을 저장한 캐시를 초기화 할 수 있다.

```kotlin
suspend fun main(): Unit = coroutineScope {
    val mutableSharedFlow = MutableSharedFlow<String>(
        replay = 2,
    )
    mutableSharedFlow.emit("Message1")
    mutableSharedFlow.emit("Message2")
    mutableSharedFlow.emit("Message3")

    println(mutableSharedFlow.replayCache)   // [Message2, Message3]

    launch {
        mutableSharedFlow.collect { println("#1 received $it") }
        // #1 received Message2
        // #1 received Message3
    }

    delay(100)
    mutableSharedFlow.resetReplayCache()
    println(mutableSharedFlow.replayCache) // []
}
```
코틀린에서는 감지만 하는 인터페이스와 변경하는 인터페이스를 구분하는 것이 관행이다.
- 예시, SendChannel, ReceiveChannel, Channel로 구분 하는 것
- 같은 법칙으로 MutableSharedFlow는 SharedFlow와 FlowCollector 모두를 상속한다.
    - SharedFlow: Flow를 상속하고 감지하는 목적 (collect)
    - FlowCollector: 값을 내보내는 목적 (emit)

```kotlin
suspend fun main(): Unit = coroutineScope {
    val mutableSharedFlow = MutableSharedFlow<String>()
    val sharedFlow: SharedFlow<String> = mutableSharedFlow
    val collector: FlowCollector<String> = mutableSharedFlow

    launch {
        mutableSharedFlow.collect { println("#1 received $it") }
    }
    launch {
        sharedFlow.collect { println("#2 received $it") }
    }

    delay(1000)
    mutableSharedFlow.emit("Message1")
    collector.emit("Message2")
}
// (1 sec)
// #1 received Message1
// #2 received Message1
// #1 received Message2
// #2 received Message2
```

안드로이드에서 SharedFlow를 사용하는 예시
```kotlin
class UserProfileViewModel {
    private val _userChanges = MutableSharedFlow<UserChange>()
    val userChanges: SharedFlow<UserChange> = _userChanges

    fun onCreate() {
        viewModelScope.launch {
            userChanges.collect(::applyUserChange)
        }
    }

    fun onNameChanged(newName: String) {
        // ...
        _userChanges.emit(NameChange(newName))
    }

    fun onPublicKeyChanged(newPublicKey: String) {
        // ...
        _userChanges.emit(PublicKeyChange(newPublicKey))
    }
}
```

## shareIn

- 플로우는 변화를 감지할 때 주로 사용 된다.
- 다양한 클ㄹ래스가 변화를 감지하는 상황에서 하나의 플로우로 여러 개의 플로우를 만들고 싶다면, SharedFlow가 해결책이다.
- Flow를 SharedFlow로 바꾸는 쉬운 방법은 shareIn 함수를 사용하는 것이다.

```kotlin
suspend fun main(): Unit = coroutineScope {
    val flow = flowOf("A", "B", "C")
        .onEach { delay(1000) }

    val sharedFlow: SharedFlow<String> = flow.shareIn(
        scope = this,
        started = SharingStarted.Eagerly,
        // replay = 0 (default)
    )

    delay(500)
    launch { sharedFlow.collect { println("#1 $it") } }

    delay(1000)
    launch { sharedFlow.collect { println("#2 $it") } }

    delay(1000)
    launch { sharedFlow.collect { println("#3 $it") } }
}
// (1 sec)
// #1 A
// (1 sec)
// #1 B
// #2 B
// (1 sec)
// #1 C
// #2 C
// #3 C
```

- shareIn 함수는 SharedFlow를 만들고 Flow의 원소를 보냅니다.
- 플로우의 원소를 모으는 코루틴을 시작하므로 shareIn 함수는 첫 번째 인자로 코루틴 스코프를 받는다.
- 세 번째 인자는 기본값이 0인 replay 값이다.
- 두 번째 인자인 started는 리스너의 수에 따라 값을 언제부터 감지할지 결정한다.
    - SharingStarted.Eagerly: 즉시 값을 감지하기 시작하고 플로울로 값을 전송한다.
    - SharingStarted.Lazily: 첫 번째 구독자가 나올 때 감지하기 시작한다. 모든 구독자가 사라져도 업스트림(데이터를 방출하는) 플로우는 Active 사태이지지만, 구독자가 없으면 replay 수 만큼 가장 최근의 값들을 캐싱한다.
    - WhileSubscribed(): 첫 번째 구독자가 나올때 감지하기 시작하며, 마지막 구독자가 사라지면 플로우도 멈춘다.
    - SharingStarted 인터페이스를 구현하여 커스텀화도 가능

```kotlin
suspend fun main(): Unit = coroutineScope {
    val flow = flowOf("A", "B", "C")

    val sharedFlow: SharedFlow<String> = flow.shareIn(
        scope = this,
        started = SharingStarted.Eagerly,
    )

    delay(100)
    launch { sharedFlow.collect { println("#1 $it") } }
    print("Done")
}
// (0.1 sec)
// Done
```

```kotlin
suspend fun main(): Unit = coroutineScope {
    val flow1 = flowOf("A", "B", "C")
    val flow2 = flowOf("D")
        .onEach { delay(1000) }

    val sharedFlow = merge(flow1, flow2).shareIn(
        scope = this,
        started = SharingStarted.Lazily,
    )

    delay(100)
    launch { sharedFlow.collect { println("#1 $it") } }

    delay(1000)
    launch { sharedFlow.collect { println("#2 $it") } }
}
// (0.1 sec)
// #1 A
// #1 B
// #1 C
// (1 sec)
// #2 D
// #1 D
```

```kotlin
suspend fun main(): Unit = coroutineScope {
    val flow = flowOf("A", "B", "C", "D")
        .onStart { println("Started") }
        .onCompletion { println("Finished") }
        .onEach { delay(1000) }

    val sharedFlow = flow.shareIn(
        scope = this,
        started = SharingStarted.WhileSubscribed(),
    )

    delay(3000)
    launch { println("#1 ${sharedFlow.first()}") }
    launch { println("#2 ${sharedFlow.take(2).toList()}") }

    delay(3000)
    launch { println("#3 ${sharedFlow.first()}") }
}
// (3 sec)
// Started
// (1 sec)
// #1 A
// (1 sec)
// #2 [A, B]
// Finished

// (1 sec)
// Started
// (1 sec)
// #3 A
// Finished
```

- 동일한 변화를 감지하려고 하는 서비스가 여러 개일 때, shareIn 을 사용하면 된다.
- 다양한 서비스가 **_location_** 에 의존하고 있다면, 각 서비스가 데이터베이스를 독자적으로 감지하는 건 최적화된 방법이 아니다.
- 대신 이런 변화를 감지하고 SharedFlow를 통해 감지된 변화를 공유하는 것이 좋다. 이때 shareIn 을 사용한다.
    - replay = 1: 구독자가 **_location_** 의 마지막 목록을 즉시 받기 원할 때
    - replay = 0: 구독 후의 변경에만 반응하려면 0으로 설정

```kotlin
@Dao
interface LocationDao {
    @Insert(onConflict = OnConflictStrategy.IGNORE)
    suspend fun insertLocation(location: Location)

    @Query("DELETE FROM location_table")
    suspend fun deleteLocations()

    @Query("SELECT * FROM location_table ORDER BY time")
    fun observeLocations(): Flow<List<Location>>
}

class LocationService(
    locationDao: LocationDao,
    scope: CoroutineScope
) {
    private val locations = locationDao.observeLocations()
        .shareIn(
            scope = scope,
            started = SharingStarted.WhileSubscribed(),
        )

    fun observeLocations(): Flow<List<Location>> = locations
}
```

> 호출 할 때마다 새로운 SharedFlow를 만들면 안되고, SharedFlow를 만든 뒤 프로퍼티로 저장하자.

# StateFlow
- StateFlow 는 SharedFlow의 개념을 확장 시킨 것으로, replay 값이 1인 SharedFlow와 비슷하게 작동한다.
- StateFlow 는 value 프로퍼티로 접근 가능한 값 하나를 항상 가지고 있다.
    - 초기 값은 생성자를 통해 전달되어야 한다.
    - value 프로퍼티로 값을 얻어 올 수 도 있고, 설정 할 수도 있다.
    - MutableStateFlow는 값을 감지할 수 있는 보관소 이다.

```kotlin
interface StateFlow<out T> : SharedFlow<T> {
    val value: T
}

interface MutableStateFlow<T> :
    StateFlow<T>, MutableSharedFlow<T> {

    override var value: T

    fun compareAndSet(expect: T, update: T): Boolean
}
```

```kotlin
suspend fun main() = coroutineScope {
    val state = MutableStateFlow("A")
    println(state.value) // A
    launch {
        state.collect { println("Value changed to $it") }
        // Value changed to A
    }

    delay(1000)
    state.value = "B" // Value changed to B

    delay(1000)
    launch {
        state.collect { println("and now it is $it") }
        // and now it is B
    }

    delay(1000)
    state.value = "C" // Value changed to C and now it is C
}
```

- 안드로이드에서 StateFlow는 LiveData를 대체하는 최신 방식으로 사용되고 있다.
    - 코루틴을 완벽하게 지원한다.
    - 초기 값을 가지고 있기 때문에 null일 필요가 없다.
    - StateFlow 는 주로 ViewModel의 상태를 나타낼 때 주로 사용된다.
    - StateFlow 는 상태를 감지할 수 있으며, 감지된 상태에 따라 뷰가 보여주고 갱신된다.

```kotlin
class LatestNewsViewModel(
    private val newsRepository: NewsRepository
) : ViewModel() {
    private val _uiState = MutableStateFlow<NewsState>(LoadingNews)
    val uiState: StateFlow<NewsState> = _uiState

    fun onCreate() {
        scope.launch {
            _uiState.value = NewsLoaded(newsRepository.getNews())
        }
    }
}
```

- StateFlow 는 데이터가 덮어 씌워지기 때문에, observe가 느린 경우 상태의 중간 변화를 받을 수 없는 경우도 있다.
    - 모든 이벤트를 다 받으려면 SharedFlow를 사용해야 한다.
    - StateFlow는 현재 상태만 나타내기 때문에, StateFlow의 이전 상태는 아무도 관심이 없다. 이는 의도적으로 설계되었다.

```kotlin
suspend fun main(): Unit = coroutineScope {
    val state = MutableStateFlow('X')
    launch {
        for (c in 'A'..'E') {
            delay(300)
            state.value = c
            // or state.emit(c)
        }
    }

    state.collect {
        delay(1000)
        println(it)
    }
}
// X
// C
// E
```

## stateIn
- stateIn 는 Flow<T>를 StateFlow<T> 로 변환하는 함수이다.
- 스코프에서만 호출 가능하지만, 중단 함수 이기도 하다.
- StateFlow는 항상 값을 가져야 하기에, 값을 명시하지 않았을 때는 첫 번째 값이 계산될 때까지 기다려야 한다.

```kotlin
suspend fun main() = coroutineScope {
    val flow = flowOf("A", "B", "C")
        .onEach { delay(1000) }
        .onEach { println("Produced $it") }
    val stateFlow: StateFlow<String> = flow.stateIn(this)

    println("Listening")
    println(stateFlow.value)
    stateFlow.collect { println("Received $it") }
}
// (1 sec)
// Produced A
// Listening
// A
// Received A
// (1 sec)
// Produced B
// Received B
// (1 sec)
// Produced C
// Received C
```

- stateIn의 두 번째 형태는 중단 함수가 아니지만, 초기 값과 started 모드를 지정해야 한다.
    - started 모드는 shareIn과 같은 옵션을 가진다.

```kotlin
suspend fun main() = coroutineScope {
    val flow = flowOf("A", "B")
        .onEach { delay(1000) }
        .onEach { println("Produced $it") }

    val stateFlow: StateFlow<String> = flow.stateIn(
        scope = this,
        started = SharingStarted.Lazily,
        initialValue = "Empty"
    )

    println(stateFlow.value)

    delay(2000)
    stateFlow.collect { println("Received $it") }
}
// Empty
// (2 sec)
// Received Empty
// (1 sec)
// Produced A
// Received A
// (1 sec)
// Produced B
// Received B
```

- 하나의 데이터 소스에서 값이 변경된 걸 감지하는 경우에 주로 stateIn 함수를 사용한다.
- StateFlow로 상태를 변경할 수 있으며, 결국엔 View가 변화를 감지 할 수 있게 된다.

```kotlin
class LocationsViewModel(locationService: LocationService) : ViewModel() {

    private val location = locationService.observeLocations()
        .map { it.toLocationsDisplay() }
        .stateIn(
            scope = viewModelScope,
            started = SharingStarted.Lazily,
            initialValue = LocationsDisplay.Loading,
        )
    // ...
}
```

# Reference
- https://kotlinlang.org/docs/coroutines-guide.html
- 코틀린 코루틴 Kotlin Coroutines: Deep Dive (Marcin Moskała, 인사이트)