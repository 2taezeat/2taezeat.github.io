---
layout: post
title: "Navigation과 Fragment 생명주기"
date: 2023-08-30 14:54:46+0900
author: 2taezeat
toc: true
categories:

- blog
- Android

---
Jetpack Navigation 사용시에, Fragment 생명주기 에 대해 알아보자
<!--more-->

> **Library 기준**
>
> androidx.navigation:navigation-fragment-ktx:2.5.3
>
> androidx.navigation:navigation-ui-ktx:2.5.3
>

# FragmentManager 를 통한 Transaction

- **add**: 호스트 Activity의 생명주기에 Fragment 생명주기 추가, add된 Fragment는 `onAttach ~ onResume`까지 호출
    - _Add a fragment to the activity state._
- **remove**: `onPause ~ onDetach`까지 호출, Fragment가 메모리에서 제거됨.
    - _Remove an existing fragment. If it was added to a container, its view is also removed from that container._
- **replace**: replace() 함수 인자로 지정된 Fragment를 제외한 나머지 모든 프래그먼트가 remove (나머지 Fragment는 `onDetach` 까지 호출, 지정된
  Fragment는 `onAttach ~ onResume` )
- **show / hide**: 기본적으로 `add`된 Fragment를 대상으로 View를 보이게 하거나 감춤.(`visibility` 변경)
- **attach**: 대상 Fragment의 `onCreateView ~ onResume` 까지 호출
    - _Detach the given fragment from the UI. This is the same state as when it is put on the back stack: the fragment
      is removed from the UI_
- **detach**: 대상 Fragment의 `onPause ~ onDestroyView` 까지 호출
    - _Re-attach a fragment after it had previously been detached from the UI with detach(android.app.Fragment). This
      causes its view hierarchy to be re-created, attached to the UI, and displayed._

# Jetpack Navigation 의 장점

- Fragment의 관계를 resource.xml 파일로 한눈에 볼 수 있다.
- `FragmentManager`를 통한 Transaction 처리에 쓰이는 코드를 줄일 수 있다.
- Fragment 간에 data 전달을 `safe-args`를 통해 할 수 있다.
- `deep link` 처리

# navigate 시 Fragment 생명주기

`navController`의 `backQueue` 변수로 `backStack`을 확인할 수 있다.

```kotlin
// navController.kt 내부 코드

/**
 * Retrieve the current back stack.
 * @return The current back stack.
 */
@get:RestrictTo(RestrictTo.Scope.LIBRARY_GROUP)
public open val backQueue: ArrayDeque<NavBackStackEntry> = ArrayDeque()
```

`navigate()` 함수 호출시, `FragmentTransaction`의 `replace()`가 호출된다.

```kotlin
// FragmentNavigator.kt 내부 코드
@Navigator.Name("fragment")
public open class FragmentNavigator(
    private val context: Context,
    private val fragmentManager: FragmentManager,
    private val containerId: Int
) : Navigator<Destination>() {
    // ... 

    private fun createFragmentTransaction(
        entry: NavBackStackEntry,
        navOptions: NavOptions?
    ): FragmentTransaction {
        val destination = entry.destination as Destination
        //..
        val frag = fragmentManager.fragmentFactory.instantiate(context.classLoader, className)
        frag.arguments = args
        val ft = fragmentManager.beginTransaction()
        //..
        ft.replace(containerId, frag)
        //..
        return ft
    }
}
```

## 상황 설정

- `bottomMenu`: GameFragment, SettingFragment 이고,
- `action`: SettingFragment -> WebViewFragment 인 상황
- GameFragment -> SettingFragment -> WebViewFragment ->(뒤로가기 버튼) SettingFragment 의 경우

{% include img_assets_400.html id="/blog/2023/08-30/bottomNavi.png" %}

## Logging 결과

- `backStack` 에서 없어지면(pop되면), `onDestoryView ~ onDetach` 까지 호출
- `backStack` 에 존재하면, `onDestoryView` 까지만 호출, 다시 `backStack` 최상단에 특정 Fragment가 존재하면, `onCreateView` 부터 호출
- A -> B로 `naviagte()` 시, B가 Create 되고, A가 Destory 됨. (선 Create, 후 Destory)

```
13:43:37.780 GameFragment         onCreate
13:43:37.802 GameFragment         onCreateView
13:43:37.802 GameFragment         [com.k031.fruitcardgame:id/game_nav_graph, com.k031.fruitcardgame:id/GameFragment]
13:43:39.586 SettingFragment      onCreate
13:43:39.586 SettingFragment      onCreateView
13:43:39.586 SettingFragment      [com.k031.fruitcardgame:id/game_nav_graph, com.k031.fruitcardgame:id/GameFragment, com.k031.fruitcardgame:id/SettingFragment]
13:43:39.740 GameFragment         onDestroyView
13:43:42.369 WebViewFragment      onCreate
13:43:42.369 WebViewFragment      onCreateView
13:43:42.369 WebViewFragment      [com.k031.fruitcardgame:id/game_nav_graph, com.k031.fruitcardgame:id/GameFragment, com.k031.fruitcardgame:id/SettingFragment, com.k031.fruitcardgame:id/WebViewFragment]
13:43:42.560 SettingFragment      onDestroyView
13:43:44.910 SettingFragment      onCreateView
13:43:44.910 SettingFragment      [com.k031.fruitcardgame:id/game_nav_graph, com.k031.fruitcardgame:id/GameFragment, com.k031.fruitcardgame:id/SettingFragment]
13:43:44.919 WebViewFragment      onDestroyView
13:43:44.920 WebViewFragment      onDestroy
```

# navigation popup

`a -> b -> c -> a` 로 group 연결

{% include img_assets_600.html id="/blog/2023/08-30/a_b_c_navi.png" %}

```xml

<fragment
        android:id="@+id/c"
        android:name="com.example.myapplication.C"
        android:label="fragment_c"
        tools:layout="@layout/fragment_c">

    <action
            android:id="@+id/action_c_to_a"
            app:destination="@id/a"

            app:popUpTo="@+id/a"
            app:popUpToInclusive="true"/>
</fragment>
```

## case 별 backStack 현황

- `... -> a -> b -> c` 까지, backStack 현황 = `[..., a, b, c]`
- app:popUpTo="@+id/a" 가 없다면, a -> b -> **c -> a** 까지, backStack 현황 = `[..., a, b, c, a]`
- app:popUpTo="@+id/a" 가 있다면, a -> b -> **c -> a** 까지, backStack 현황 = `[..., a, a]`
- app:popUpTo="@+id/a" 있고, app:popUpToInclusive="false" 이면, a -> b -> **c -> a** 까지, backStack 현황 = `[..., a, a]`
- app:popUpTo="@+id/a" 있고, app:popUpToInclusive="true" 이면, a -> b -> **c -> a** 까지, backStack 현황 = `[..., a]`

## 결론

- `popUpTo` 속성으로 특정 Fragment를 지정하면, 그 사이의 `NavBackStackEntry`은 backStack에서 제거된다.
- `popUpToInclusive` 속성을 `true`로 설정해주면, backStack `popUpTo`로 지정된 `NavBackStackEntry`은 제거된다.
- `popUpToInclusive` 값을 지정하지 않으면, `popUpToInclusive=false`와 같다.

# Reference

- [https://developer.android.com/reference/android/app/FragmentTransaction#attach(android.app.Fragment)](https://developer.android.com/reference/android/app/FragmentTransaction#attach(android.app.Fragment))
  .
- [https://developer.android.com/guide/navigation/navigation-navigate?hl=ko#pop](https://developer.android.com/guide/navigation/navigation-navigate?hl=ko#pop)
  .
- [https://velog.io/@dabin/%EC%95%88%EB%93%9C%EB%A1%9C%EC%9D%B4%EB%93%9CFragment-2%ED%8E%B8FragmentR#framgment-%EA%B5%90%EC%B2%B4replace%EC%9E%91%EC%97%85](https://velog.io/@dabin/%EC%95%88%EB%93%9C%EB%A1%9C%EC%9D%B4%EB%93%9CFragment-2%ED%8E%B8FragmentR#framgment-%EA%B5%90%EC%B2%B4replace%EC%9E%91%EC%97%85)
  .