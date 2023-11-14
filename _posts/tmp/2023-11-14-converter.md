
- 작성자 : [2taezeat](https://github.com/2taezeat)
- 작성일자 : 23.11.14(화)

---
- [개요](#개요)

- [Gson vs Moshi](#gson-vs-moshi)
  * [Gson](#gson)
  * [Moshi](#moshi)
  * [Moshi 가 Gson 보다 더 좋은 점](#Moshi-가-Gson-보다-더-좋은-점)

- [kotlinx.serialization 의 대표적인 특징](#kotlinx.serialization-의-대표적인-특징)
  * [Kotlin-oriented](#kotlin-oriented)
  * [compile-time safe](#compile-time-safe)
  * [Polymorphic serialization](#polymorphic-serialization)
- [결론](#결론)
- [reference](#reference)

**여러 역/직렬화 라이브러리 중 무엇을 쓸까?**

---

# 개요
- android application 에서는 주로 `retrofit2` 을 사용하여, json(JavaScript Object Notation) 을 주고 받으면서 서버와 통신한다.

- json 직렬화(Serialization)란, 프로그래밍 언어에서 사용되는 데이터 구조를 json 형식으로 변환하는 과정

- 역직렬화 : 직렬화된 데이터를 다시 객체의 형태로 만드는 것

- retrofit 를 이용하여 서버와 통신할 때, converter를 지정해주면 retrofit이 자동으로 역/직렬화 를 수행해준다.

- 이때, 여러 converter, 역/직렬화 라이브러리 들이 있다.
  - 대표적으로 `Gson`, `Moshi`, `kotlinx.serialization` 이 존재한다.

- 3가지 라이브러리들을 비교 해보고 `kotlinx.serialization`를 선택한 이유와 특징을 살펴본다.


# Gson vs Moshi
## Gson
- Google에서 개발한 Java 기반의 오픈 소스 json 데이터 처리 라이브러리
- Gson은 현재 maintenance mode로 새로운 기능을 개발하는 것이 아닌 bug fix와 같은 유지보수성 업데이트만 진행되고 있다.

## Moshi
- retrofit을 만든, Square사에서 개발한 json 직렬화 라이브러리로, java와 kotlin에서 JSON 데이터를 처리하는 데 사용된다.

## Moshi 가 Gson 보다 더 좋은 점
- 높은 성능과 속도: Moshi는 경량화된 디자인과 최적화된 구조를 통해 높은 성능과 빠른 속도를 지원합니다.

- Kotlin 호환성: Moshi는 Kotlin과의 높은 호환성을 가지고 있습니다. Kotlin의 Nullable 형식을 자동으로 처리하여 개발자가 간편하게 JSON 데이터를 다룰 수 있도록 지원합니다.

- 정적 타입 지원: Moshi는 코드 생성(codegen)을 통해 정적 타입 어노테이션을 지원합니다. 이를 활용하여 JSON 데이터의 파싱 과정에서 타입 안정성을 제공하고, 컴파일 시간에 발생할 수 있는 오류를 미리 방지할 수 있습니다.

- 코드 자동 생성(codegen): Moshi는 어노테이션 기반 코드 생성을 지원하여 JSON 데이터 처리를 런타임이 아닌 컴파일 시점에 처리할 수 있도록 도와줍니다. 이로 인해 더 빠른 성능과 개발 생산성을 제공합니다.


# kotlinx.serialization 의 대표적인 특징
- kotlin 1.4 version 에서 나온 JetBrain이 개발한 역/직렬화 라이브러리
- 다른 컨버터 라이브러리와 다르게 Reflection을 사용하지 않고 개발한 라이브러리 (성능 이점)

## Kotlin-oriented
- kotlin의 `default value` 기능을 사용할 수 있다.
- KMM, kotlin/js, kotlin/native 에서도 사용가능하다.

```kotlin
@Serializable
data class Project(
  val name: String,
  val language: String = "Kotlin"
)
```
이 문자열을 Gson 라이브러리로 파싱하면, name에는 정상적으로 값이 대입되지만 language에는 null이 대입된다.
즉, null이 아닌 프로퍼티가 null 값을 가지게 될 뿐 아니라, language 프로퍼티의 기본값 조차 반영되지 않는 문제가 발생한다.

```kotlin
println(Gson().fromJson(inputString, Project::class.java))
//Gson: Project(name=kotlinx.serialization, language=null) 

println(Json.decodeFromString<Project>(inputString))
//kotlinx.serialization: Project(name=kotlinx.serialization, language=Kotlin)
```


## compile-time safe
- 기본적으로 `@Serializable` 이 있는 클래스만 직렬화한다.
  - 직렬화를 수행할 수 없는 경우 런타임 에러 대신 컴파일 에러가 발생하므로 버그를 사전에 방지가 가능하다.

## Polymorphic serialization
- 상속 구조나 다형성을 파악해 직렬화를 해준다.

```kotlin
@Serializable
sealed class Project {
  abstract val name: String
  var status = "open"
}

@Serializable
@SerialName("owned")
class OwnedProject(override val name: String, val owner: String) : Project()

fun main() {
  val json = Json { encodeDefaults = true } // "status" will be skipped otherwise
  val data: Project = OwnedProject("kotlinx.coroutines", "kotlin")
  println(json.encodeToString(data))
}
// {"type":"owned","status":"open","name":"kotlinx.coroutines","owner":"kotlin"}

```

# 결론
Moshi이 Gson에 비해 장점도 있지만, kotlinx.serialization 의 대표적인 특징 때문에,
retrofit의 역/직렬화 converter 라이브러리는 kotlinx.serialization 으로 선택하였다.


# reference

- https://www.youtube.com/watch?v=Azi57n59ICM
- https://proandroiddev.com/goodbye-gson-hello-moshi-4e591116231e
- https://blog.imqa.io/json-moshi/
- https://www.androidhuman.com/2020-11-08-kotlin_1_4_serialization
- https://github.com/Kotlin/kotlinx.serialization