---
layout: post
title: Kotlin Koin으로 안드로이드 의존성 주입 해보기
tags:
  - kotlin
  - koin
  - android
  - di
---

```kotlin
val photoListModule: Module = module {
    factory {
        PhotoPresenterImpl(get()) as PhotoPresenter
    }
}
```
