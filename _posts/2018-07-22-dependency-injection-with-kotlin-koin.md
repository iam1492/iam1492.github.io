---
title: Kotlin 안드로이드 의존성 주입 라이브러리 Koin을 소개합니다.
layout: single
category: post
comments: true
author_profile: true
tags:
  - kotlin
  - koin
  - android
  - di
---

안드로이드 앱을 개발할때 [Dagger](https://google.github.io/dagger/) 와 같은 [Dependency Injection](https://ko.wikipedia.org/wiki/%EC%9D%98%EC%A1%B4%EC%84%B1_%EC%A3%BC%EC%9E%85)(이하 DI)을 사용하는 것을 종종 볼 수 있습니다. 각 컴포넌트간의 의존성을 외부 컨테이너에서 관리하는 방식을 통해 코드 재사용성을 높이고 Unit Test도 편하게 할 수 있게 되는 장점을 가지고 있습니다.

그런데 최근에는 코틀린이 안드로이드 앱 개발 공식 언어가 되면서 코틀린의 다양한 언어적 장점을 이용한 DI 라이브러리가 생겨났고 그 중 하나인 [Koin](https://beta.insert-koin.io/) 라이브러리에 알아보려고 합니다. 
	
> 포스팅은 최근 릴리즈한 Koin 1.0.0 beta 버전을 사용했습니다.

사실 엄밀히 따져말하면 Koin은 DI 라이브러리라기 보단 Service Locator Pattern의 구현체라고 봐야합니다. Service Locator Pattern 과 Dependency Injection Pattern은 약간의 차이가 있으며 개발자들간에 이를 바라보는 시각도 다양합니다. 이와 관련된 온라인에서의 토론들도 있는데 한번씩 훑어보면 이해하기 더 수월할 것 같습니다. JakeWharton과 코인 개발자가 댓글에 등장하니 꼭 한번 읽어보세요 :)
* [Dagger2 Vs Koin for dependency injection?](https://www.reddit.com/r/androiddev/comments/8ch4cg/dagger2_vs_koin_for_dependency_injection/)
* [Service Locator is an Anti-Pattern](http://blog.ploeh.dk/2010/02/03/ServiceLocatorisanAnti-Pattern/)
* [What's the difference between the Dependency Injection and Service Locator patterns?
](https://stackoverflow.com/questions/1557781/whats-the-difference-between-the-dependency-injection-and-service-locator-patte)

한가지 명확한 사실은 Dagger를 실제 프로젝트에 적용하기에 어느정도 학습 비용이 든다는 사실이고, Koin의 개발자는 이런 부분에 가장 촛점을 맞춰 사용하기 쉬우면서 어느정도는 DI가 주는 장점을 제공할 수 있는 라이브러리를 만든게 아닌가 싶습니다. 

아주 간단한 예제를 통해 어떻게 Koin 라이브러리로 DI을 구현하게 되는지 살펴보겠습니다. 만들어볼 예제는 [jsonplaceholder](https://jsonplaceholder.typicode.com) 에서 제공하는 sample api를 사용하였고 샘플 사진 한장을 요청해 화면에 뿌려주는 간단한 앱입니다.

## 초기화

Koin의 초기화는 정말 간단합니다. Application을 상속받은 클래스에 startKoin 함수를 호출하면서 미리 정의한 module들을 인자로 넘깁니다.

```kotlin
class KoinPhotoApp: Application(){
    override fun onCreate() {
        super.onCreate()

        startKoin(this, appModules)
    }
}
```

## 모듈 정의

Koin에서는 오직 필요한 모듈들을 정의하고 필요한 곳에 by inject() 키워드를 통해 의존성을 주입하면 됩니다. 추가적인 component 나 subcomponent들은 필요없습니다. 
Koin은 모듈을 정의할때 kotlin dsl를 사용하여 좀 더 직관적으로 정의가 가능합니다.

Koin에서 사용하는 dsl 키워드는 총 5가지가 있습니다.
* module - Koin모듈을 정의할때 사용
* factory - Dagger에서의 ActivityScope, FragmentScope와 유사한 기능으로 inject하는 시점에 해당 객체를 생성
* single - Dagger에서늬 Singleton 과 동일하며 앱이 살아있는 동안 전역적으로 사용가능한 객체를 생성
* bind - 생성할 객체를 다른 타입으로 바인딩하고 싶을때 사용
* get - 주입할 각 컴포넌트끼리의 의존성을 해결하기 위해 사용합니다.

잘이해가 안되더라도 예제를 보면 쉽게 이해할 수 있으니 우선 넘어가겠습니다.
그럼 이제 api를 호출하기 위한 retrofit service 객체를 모듈에 정의하고 위에 startKoin 인자에 넘길 appModules array를 생성해 보겠습니다.

```kotlin
val apiModule: Module = module {

    single {
        Retrofit.Builder()
             .client(OkHttpClient.Builder().build())
             .addCallAdapterFactory(RxJava2CallAdapterFactory.create())
             .addConverterFactory(GsonConverterFactory.create())
             .baseUrl(getProperty<String>("BASE_URL"))
             .build()
             .create(Api::class.java)
    }
}

val photoListModule: Module = module {
    factory {
        PhotoPresenterImpl(get()) as PhotoPresenter
    }
}

val appModules = listOf(apiModule, photoListModule)
```

코틀린은 이런 경우 따로 class 를 만들필요가 없기 때문에 위와 같이 단순히 모듈들을 변수로 정의했습니다.
여기에서 single, factory 키워드를 사용해 객체를 생성했습니다. 이렇게 생성할 경우 Retrofit api 객체는 전역적으로 한개의 객체만 생성이 가능하며 PhotoPresenter 객체는 생성한 액티비티 혹은 프래그먼트로 scope가 제한됩니다.

아래는 동일한 기능을 하는 Dagger의 모듈 구현부입니다.
```kotlin
@Provides
@Singleton
fun provideApi(): Api {
    return Retrofit.Builder()
        .client(OkHttpClient.Builder().build())
        .addCallAdapterFactory(RxJava2CallAdapterFactory.create())
        .addConverterFactory(GsonConverterFactory.create())
        .baseUrl(baseUrl)
        .build()
        .create(Api::class.java)
}
```

위에 Koin모듈 정의를 보면 get() 키워드를 사용하고 있습니다. 여기서 get()을 호출하면 apiModule 에 정의된 retrofit Api 클래스 객체가 넘어갑니다. Koin은 생성하고자 하는 객체의 인자 타입을 보고 의존성을 판단해 쉽게 컴포넌트간의 의존성을 해결합니다. 실제로 PhotoPresenterImpl의 클래스 정의는 아래와 같습니다.

```kotlin
class PhotoPresenterImpl(val api: Api): PhotoPresenter() {
    override fun requestPhoto(id: Long) {
        //구현체
    }
}
```

좀 더 쉽게 설명하자면 아래와 같은 의존성이 있는 클래스들이 있다면 

```kotlin
class ComponentA()
class ComponentB(val componentA : ComponentA)
```

모듈 정의를 아래와 같이 하게 됩니다.
```kotlin
val moduleA = module {
    // Singleton ComponentA
    single { ComponentA() }
}

val moduleB = module {
    // Singleton ComponentB with linked instance ComponentA
    single { ComponentB(get()) }
}
```

## 의존성 주입

이제 사용한 모듈들을 정의했으니 사용하고 싶은 곳에 주입해보겠습니다.
Koin에서 주입은 by inject() 키워드를 사용(Kotlin Delegated Properties 방식을 사용)합니다. 이는 Dagger에서의 @Inject 키워드와 유사하며 항상 지연초기화(lazy init)을 사용하게 됩니다. 즉 객체를 사용하는 시점에 생성을 하므로 성능상 이점이 있습니다.

```kotlin
class PhotoActivity : AppCompatActivity(), PhotoScene {

    private val presenter: PhotoPresenter by inject()

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_photo)

        presenter.scene = this
        presenter.requestPhoto(1)
    }
    ...
}
```

예제에서는 PhotoPresenter 인터페이스 객체를 주입시키고 있고 실제 객체가 생성되는 시점은 presenter.scene을 호출하는 시점입니다.
모듈을 정의하다보면 가끔 같은 객체를 여러개 생성해 각각 다른 목적으로 사용하고 싶을때가 있습니다.

Koin 에서는 모듈의 이름을 정의해 이와 같은 conflict 를 해결합니다.

```kotlin
val dialogModule: Module = module {
    module("cancelableDialogBuilder"){
        single {
            AlertDialog.Builder(androidContext())
            .setCancelable(true)
        }
    }
    module("dialogBuilder"){
        single {
            AlertDialog.Builder(androidContext())
                .setCancelable(false)
        }
    }
}
```

```kotlin
// Request dependency from namespace
val cancelableDialog: AlertDialog.Builder by inject("cancelableDialogBuilder")
```

실제 프로젝트에서는 이렇게 사용하지는 않겠지만 이해를 돕기 위해 AlertDialog 사용해봤습니다.


여기까지 모듈을 정의하고 정의된 모듈을 주입하는것까지 해보았습니다. 이 외에 ViewModel 주입을 더 편하게 해주는 [koin-android-viewmodel](https://beta.insert-koin.io/docs/1.0/documentation/koin-android/index.html#_architecture_components_with_koin_viewmodel) 라이브러리도 있고 Scoping을 편하게 해주는 [koin-android-scope](https://beta.insert-koin.io/docs/1.0/documentation/koin-android/index.html#_scope_features_for_android) 라이브러리도 제공을 하고 있습니다.
Dagger와 비교했을때 실제 학습비용은 매우 낮지만 장단점이 있습니다. 판단은 각자의 몫이지만 좀더 가벼운 프로젝트를 시작할때 한번쯤 사용해보는것도 좋을 것 같습니다.

모든 프로젝트는 github에 [KoinSample](https://github.com/daangn/KoinSample) 프로젝트에 올려두었습니다.



