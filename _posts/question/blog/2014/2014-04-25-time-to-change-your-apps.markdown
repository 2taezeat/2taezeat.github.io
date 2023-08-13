---
layout: post
title: "Time to change your apps"
date: 2014-04-25 16:54:46
author: Admin
categories: 
- blog 
- Wordpress
- Photoshop
---
hello123123
<!--more-->

> 샘플 코드 : https://github.com/Pluu/VersionCodeUpdaterSample

## 방법 1) defaultConfig 속성 수정

```kotlin
// app/build.gradle.kts
android {
  defaultConfig {
    versionCode = 1 // 적용되는 Version Code
  }
}
```

## 방법 2) Flavor를 통해서 정의

```kotlin
// app/build.gradle.kts
android {
  defaultConfig {
    versionCode = 1
  }

  flavorDimensions += "version"
  productFlavors {
    create("demo") {
      dimension = "version"
      versionCode = 1234 // 최종 적용되는 Version Code
    }
  }
}
```

## 방법 3) androidComponents의 onVariants를 사용

```kotlin
// app/build.gradle.kts
android {
  defaultConfig {
    versionCode = 1
  }

  flavorDimensions += "version"
  productFlavors {
    create("demo") {
      dimension = "version"
      versionCode = 3
    }
  }
}

androidComponents {
  finalizeDsl {
    it.defaultConfig.versionCode = 2
  }
  onVariants { variant ->
    variant.outputs.forEach { output ->
      output.versionCode.set(1234) // 최종 적용되는 Version Code
    }
  }
}
```

방법 4에서 적용되는 Version Code가 순서

1. android.defaultConfig.versionCode : `1` 적용
2. androidComponents.finalizeDsl : `2` 적용
3. android.productFlavors : `3` 적용
4. androidComponents.onVariants : `1234` 적용