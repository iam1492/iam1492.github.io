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

class PhotoActivity : AppCompatActivity(), PhotoScene {

    private val presenter: PhotoPresenter by inject()

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_photo)

        presenter.scene = this
        presenter.requestPhoto(1)
    }

    override fun onLoadPhoto(photo: Photo) {
        val photoView = findViewById<ImageView>(R.id.thumbnail_url)
        val title = findViewById<TextView>(R.id.title)

        title.text = photo.title
        Glide.with(this)
            .load(photo.thumbnailUrl)
            .into(photoView)
    }

    override fun onLoadFail(error: String?) {
        Toast.makeText(this, "load failed: $error", Toast.LENGTH_LONG).show()
    }

    override fun onDestroy() {
        super.onDestroy()
        presenter.dropScene(this)
    }
}
```
