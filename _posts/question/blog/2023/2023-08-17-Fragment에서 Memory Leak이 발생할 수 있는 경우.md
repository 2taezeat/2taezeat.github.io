---
layout: post
title: "Fragment에서 Memory Leak이 발생할 수 있는 경우"
date: 2023-08-17 16:54:46+0900
author: 2taezeat
toc: true
categories:
- blog
- Android

---
Fragment에서 Memory Leak이 발생할 수 있는 경우에 대해 알아보자.
<!--more-->


> **Library 기준**
>
> androidx.fragment:fragment-ktx:1.5.7
>
> androidx.recyclerview:recyclerview:1.3.0

# Fragment Lifecycle

`Fragment.java` 파일과 공식 문서 를 보면 알 수 있듯이, Fragment는 `mLifecycleRegistry` 와 `mViewLifecycleOwner` 두 개의 Lifecycle를 가지고 있다.
`Fragment 생명주기`를 View lifecycle 과 Fragment 자체 lifecycle로 분리해서 바라보면, callback 함수명에서 알 수 있듯이,
`onDestoryView()` 호출로 Fragment의 View는 사라지고, Fragment는 `onDetach()` 호출 전과 `onDestoryView()` 호출 후에 `onDestory()` 가 호출되어 사라진다.



{% include img_assets_800.html id="/blog/2023/08-17/fragmentLifecycle.png" %}


```kotlin
//Fragment.java

private void initLifecycle() {
    mLifecycleRegistry = new LifecycleRegistry (this);
    // ...
}

// ... 
@MainThread
@NonNull
public LifecycleOwner getViewLifecycleOwner() {
    // ...
    return mViewLifecycleOwner;
}

// ... 
@Override
@NonNull
public Lifecycle getLifecycle() {
    return mLifecycleRegistry;
}
```



```kotlin
void performStart () { // 예시 performStart() 함수, mLifecycleRegistry, mViewLifecycleOwner을 모두 이용하는 모습
    mChildFragmentManager.noteStateNotSaved();
    mChildFragmentManager.execPendingActions(true);
    mState = STARTED;
    mCalled = false;
    onStart();
    if (!mCalled) {
        throw new SuperNotCalledException ("Fragment " + this
                + " did not call through to super.onStart()");
    }
    mLifecycleRegistry.handleLifecycleEvent(Lifecycle.Event.ON_START);
    if (mView != null) {
        mViewLifecycleOwner.handleLifecycleEvent(Lifecycle.Event.ON_START);
    }
    mChildFragmentManager.dispatchStart();
}

```

# RecyclerView 내부 구조
{% include img_assets_600.html id="/blog/2023/08-17/fragment_structure.png" %}


`RecyclerView.java` 내부를 보면, `Adapter`는 `mObservable`을 가지고 있고, Observer들은 `RecyclerView`를 참조한다.
또한 `RecyclerView`는 `Adapter`를 참조하기에,
`Adatper`와 `RecyclerView`는 `양방향 참조, cycle`이 생긴다.

```kotlin
// RecyclerView.java

public class RecyclerView extends ViewGroup implements ScrollingView,
NestedScrollingChild2, NestedScrollingChild3 {
    //...
    private final RecyclerViewDataObserver mObserver = new RecyclerViewDataObserver();
    //...
    Adapter mAdapter;
    //...
}

private class RecyclerViewDataObserver extends AdapterDataObserver {
    // ...
}

public abstract static class Adapter<VH extends ViewHolder> {
    //...
    private final AdapterDataObservable mObservable = new AdapterDataObservable();
    //...
}

static class AdapterDataObservable extends Observable<AdapterDataObserver> {
    public boolean hasObservers() { return !mObservers.isEmpty(); }
    //...
}

```


# Memory Leak 발생 가능성

- `BottomNaviagtion`을 사용하는 상황
- `BottomNavView` 에 A,B,C 3개의 Fragment가 menu로 등록된 상태
- 현재 A_Fragment가 `Resume`인 상황에서, B_Fragment로 전환시에
  - A_Fragment는 `onDestoryView()`가 호출
  - `RecyclerView`가 사라질때, `mOberver` 참조로 인해 `Memory Leak` 발생

>View or Data `Binding` 를 사용할때도 유사하게 `Memory Leak` 이 발생 할 수 있다.
```kotlin
class A_Fragment : Fragment() {

    private lateinit var mockAdapter: MockAdapter
    
    override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
        super.onViewCreated(view, savedInstanceState)
        mockAdapter = MockAdapter()
        
        val recyclerView: RecyclerView = view.findViewById(R.id.recycler_view)
        recyclerView.layoutManager = LinearLayoutManager(this.context)
        recyclerView.adapter = mockAdapter
    }
}

```

`LeakCanary` 을 통해 확인한 `Memory Leak` 상태

{% include img_assets_800.html id="/blog/2023/08-17/memoryLeak.png" %}


# 해결 방법
**`Adatper`와 `RecyclerView`의 `양방향 참조` 를 `onDestoryView()`시에 해제**

1. mockAdapter 를 nullable 하게 설정
2. `onDestoryView()` 호출시 `mockAdapter = null`
3. B_Fragment에서 다시 A_Fragment 로 전환시, Fragment는 `onCreateView()` 부터 생명주기 다시 시작
4. `onCreateView()` 호출시에 mockAdapter를 MockAdapeter() 로 초기화

```kotlin
class A_Fragment : Fragment() {

    private var mockAdapter: MockAdapter? = null

    override fun onCreateView(
        inflater: LayoutInflater,
        container: ViewGroup?,
        savedInstanceState: Bundle?
    ): View? {
        mockAdapter = MockAdapter()
        //...
    }
    
    
    override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
        super.onViewCreated(view, savedInstanceState)
        
        val recyclerView: RecyclerView = view.findViewById(R.id.recycler_view)
        recyclerView.layoutManager = LinearLayoutManager(this.context)
        recyclerView.adapter = mockAdapter
    }

    override fun onDestroyView() {
        super.onDestroyView()
        mockAdapter = null
    }
}

```


# Reference

- [https://charlesmuchene.com/a-subtle-memory-leak-fragment-recyclerview-and-its-adapter](https://charlesmuchene.com/a-subtle-memory-leak-fragment-recyclerview-and-its-adapter).
- [https://pluu.github.io/blog/android/2020/01/25/android-fragment-lifecycle/](https://pluu.github.io/blog/android/2020/01/25/android-fragment-lifecycle/).