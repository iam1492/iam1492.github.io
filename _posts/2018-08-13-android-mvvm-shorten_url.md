---
title: Android MVVM 패턴, ViewModel, LiveData, Databinding을 이용해 간단한 Toy App 만들어보기
layout: single
category: post
comments: true
tags:
  - mvvm
  - viewmodel
  - livedata
  - android
  - databinding
---

# Model - View - ViewModel 

MVVM은 Model - View - ViewModel의 줄임말로 최근 안드로이드 아키텍쳐에 가장 많은 사랑을 받으며 구글의 전폭적인 지원을 받기도 하고 있습니다. [당근마켓](https://www.daangn.com/)의 경우에는 몇년전에 개발을 시작했던거라 당시에 트렌드였던 MVP로 구조로 개발이 되어있으나 점점 서비스 규모가 커지면서 혼자 개발함에도 불구하고 관리의 어려움이 많았고 테스트 코드에 대한 니즈가 생기면서 구조적인 변화를 고민하게 되었습니다. 그래서 최근에 Clean Architecture, MVVM 등을 찾아보면서 그 중 MVVM 관련해서 정리를 해보려고 합니다. MVVM은 말그대로 아키텍쳐 패턴입니다. MVVM이 어떻게 탄생했는지에 대해서는 [위키피디아](https://en.wikipedia.org/wiki/Model%E2%80%93view%E2%80%93viewmodel)를 참고해보세요 :)

## MVVM 구조의 장점

MVVM 패턴을 지켜 개발된 앱은 아래와 같은 특징을 갖게 됩니다.
1. [Separation of concern](https://en.wikipedia.org/wiki/Separation_of_concerns) - 하나의 소프트웨어를 최대한 기능적으로 작은 단위로 나눔 
2. 테스트가 쉬워지고 큰프로젝트도 상대적으로 관리하기 좋다.
3. [SOLID principle](https://en.wikipedia.org/wiki/SOLID) 을 지향 
4. 앱이 구조적으로 약한 결합의 컴포넌트로 나눠짐

대체적으로 위와같은 장점에 대해서 이야기를 하지만 결국 가장 큰 목적은 유지보수가 쉽고 테스트가 용이한 코드를 만드는 것입니다.

![mvvm](/assets/images/mvvm.png)

MVVM의 기본 구조를 그림으로 표현한 것입니다. View는 ViewModel에게 클릭이벤트, 필요한 데이터 요청등을 명시적으로 하고 ViewModel이 nofity할때까지 기다리게 됩니다. 그와 동일하게 ViewModel은 Model을 통해 데이터를 요청하고 기다리게 됩니다. 각각의 컴포넌트간 레퍼런스를 갖지않고 단방향(View -> ViewModel -> Model)의 디펜던시만을 갖게 됩니다.

## Android Architecture Component

구글에서 제공하는 [Android Architecture Component](https://developer.android.com/topic/libraries/architecture/)(이하 AAC)의 몇몇 라이브러리를 이용하면 MVVM 구조로 앱 개발을 더 쉽게 할 수 있습니다. MVVM 패턴 이라고 말하면 디자인 패턴중 하나이기 때문에 반드시 AAC 라이브러리를 사용할 필요는 없습니다. 하지만 AAC 라이브러리에는 ViewModel 추상클래스를 명시적으로 제공하며 이 추상클래스를 통해 lifecycle aware viewmodel를 만들 수 있게 도와줍니다.
안드로이드 개발을 하다보면 Activity나 Fragment의 복잡한 lifecycle 때문에 데이터 상태관리의 어려움을 느끼게 되지만 AAC ViewModel을 사용할 경우 lifecycle의 영향을 받지 않고 ViewModel이 가진 데이터를 안전하게 관리할 수 있습니다. 중요한 부분이기 때문에 뒤에 구현 코드를 통해 다시 설명하겠습니다.
이제부터 설명하는 ViewModel은 AAC에서 제공하는 ViewModel의 기능적인 부분을 포함해서 설명하겠습니다.

## ViewModel
1. **View와 Model 사이의 Mediator 역할을 합니다.** 즉, Model에서 제공받은 데이터를 UI에서 필요한 정보로 가공한뒤 View가 가져갈 수 있게 데이터 변경에 대한 "이벤트"를 보내게 됩니다.
2. **ViewModel과 View는 MVP패턴과는 다르게 Many to One의 관계를 가질 수 있습니다.** 즉, 여러개의 Fragment가 하나의 ViewModel을 가질 수 있습니다.
3. **ViewModel은 View에 영향을 끼칠 수 있는 Model의 상태 관리도 담당합니다.** (예: 로딩중 상태, 네트워크 에러 상태, 오프라인, visibility 등)
4. **View 또는 액티비티 Context에 대한 레퍼런스를 가지면 안됩니다.** View는 ViewModel의 reference를 가지지만 ViewModel은 View에 대한 정보가 전혀 없어야 합니다. (ViewModel이 View의 레퍼런스를 가진다면 lifecycle 에 따른 메모리릭이 발생할 수 있는데 그 이유는 ViewModel이 destroy 외의 라이프사이클에서는 메모리에서 해제되지 않기 때문입니다.)
5. 앱이 백그라운드에서 죽는 경우에는 뷰모델도 함께 사라지기 때문에 이 경우에 한해서는 onSaveInstanceState 를 통해 복구할 데이터를 저장해야합니다.

## View
1. Activity, Fragment, CustomView, Dialog, Toast, Snackbar, Menu 등의 UI 컴포넌트를 의미합니다.
2. View는 UI 업데이트를 위해 ViewModel과 바인딩이 하게 됩니다. 다른 표현으로는 View가 ViewModel에 구독을 하게되고 ViewModel의 상태가 변경되면 그 이벤트를 받아 UI를 갱신합니다.
3. 추가로 퍼미션관련된 처리, startActivity 등의 네비게이션 역할도 하게됩니다.

## Model
1. DataModel이라고도 하며 Network, DB, SharedPreference등 다양한 데이터소스로 부터 필요한 데이터를 준비합니다.

## LiveData
예제 코드를 알아보기 전에 마지막으로 AAC에 포함된 [LiveData](https://developer.android.com/topic/libraries/architecture/livedata)에 대해 간단히 살펴보겠습니다. LiveData는 "observable data holder class"로 특정 데이터를 옵져버블하게 만들게 됩니다. RxJava의 Observable과의 가장 큰 차이점은 LiveData의 경우 View lifecycle에 따라 필요한 일을 알아서 해주기 때문에 개발자가 따로 신경을 쓰지않아도 됩니다. 아래와 같은 특징을 갖습니다.
1. Lifecycle aware simple observer - 여기서 simple인 이유는 단순히 하나의 데이터에 observe를 할 수 있게만 해주기 때문입니다.
2. Activity, Fragment is LifecycleOwner - LiveData와 바인딩을 하기 위해서는 LifecycleOwner interface 구현체여야 합니다. 
3. Lifecycle 을 알기 때문에 View가 destroy되면 알아서 Observe상태를 해제합니다. (rxjava의 dispose를 lifecycle에 따라 자동으로 수행) 이런 특징은 다음의 장점이 있습니다.
    - lifecycle에 따라 알아서 observing을 해제하여 메모리릭이 안생긴다.
    - 백그라운드에서 View를 건드려 앱이 죽는 문제가 사라짐.
4. View와 ViewModel의 LiveData간의 디펜던시가 단방향으로 약한 결합을 가지게 됩니다.
5. 만약 Rxjava를 쓴다면 개발자가 lifecycle에 맞춰 dispose를 해줘야 합니다.


# Toy App - shorten Url 생성앱

이제 위의 개념이 어느정도 감이 왔다면 코드를 보면서 실제 어떻게 구현이 되는지 알아보겠습니다. 샘플로 만들 앱은 사용자가 특정 URL을 입력하면 naver api를 이용해 단축 URL을 만들어주는 단순한 앱입니다. 먼저 샘플앱에서 사용할 라이브러리를 간단히 소개해볼게요
1. Koin - [Dependency Injection](https://ko.wikipedia.org/)을 위해 사용합니다. 
2. Retrofit - Naver api 를 호출하기 위한 REST Api 라이브러리 입니다.
3. RxJava - Model에서 데이터를 notify할때 사용합니다. (Model이 Observable이라면 ViewModel은 Subscriber)
4. LiveData - ViewModel에서 View에 데이터를 notify할때 사용합니다. View는 데이터를 제공받기위해 ViewModel에 observe를 해야합니다.
5. Databinding - ViewModel과 xml layout간의 데이터를 바인딩하기 위해 사용합니다.

이 포스팅에서는 Dependency Injection에 구현방법에 대해서는 생략하겠습니다. Koin 라이브러리의 사용법이 궁금하다면 이전 포스팅인 [Kotlin 안드로이드 의존성 주입 라이브러리 Koin을 소개합니다.]({% post_url 2018-07-22-dependency-injection-with-kotlin-koin %}) 를 참고해주세요.

## Package 구조

<img src="/assets/images/packages.png" width="460" class="align-center">

패키지 구조는 이해를 돕기 위해 의도적으로 view, viewmodel, model로 나누었습니다. 
각각의 구현부분은 최대한 단순하게 만들려고 노력했습니다.

## Model
모델은 네트워크를 통해 `naver shorten url` API를 요청해 데이터를 가져오는 역할을 합니다.

```kotlin
import io.reactivex.Single

interface Repository {
    fun getShortenUrl(url: String): Single<ShortenUrl>
}
```

```kotlin
import com.daangn.www.mvvmsample.api.Api
import io.reactivex.Single

class NetworkRepositoryImpl(val api: Api): Repository {
    override fun getShortenUrl(url: String): Single<ShortenUrl> {
        return api.shorturl(url)
            .map { shortenUrlResponse ->
                ShortenUrl(
                    url = shortenUrlResponse.result.url,
                    hash = shortenUrlResponse.result.hash,
                    orgUrl = shortenUrlResponse.result.orgUrl)
            }
    }
}
```

실제 코드 구현은 매우 간단하게 되어있습니다. 여기서 ShortenUrl은 단축url 데이터를 담을 kotlin data class 입니다.
Api response spec상 몇가지 메타 데이터를 함께 내려주고 있어서 rxjava의 map을 이용해 ShortenUrl로 매핑해주는 구현이 들어가있습니다.
코드를 보면 Model의 경우 데이터를 RxJava Single을 이용해 Observable하게 만들고 있습니다. 이는 ViewModel에서 이 데이터를 observe 가능하게 만들어 위에서 설명했던 단방향의 디펜던시를 갖기 위함입니다.

## ViewModel


```kotlin
import androidx.lifecycle.ViewModel
import io.reactivex.disposables.CompositeDisposable
import io.reactivex.disposables.Disposable

open class DisposableViewModel: ViewModel() {

    private val compositeDisposable = CompositeDisposable()

    fun addDisposable(disposable: Disposable) {
        compositeDisposable.add(disposable)
    }

    override fun onCleared() {
        compositeDisposable.clear()
        super.onCleared()
    }
}
```

DisposableViewModel은 AAC의 ViewModel의 구현체이며 ShortenUrlViewModel의 상위 클래스입니다. RxJava의 경우 lifecycle을 알지 못하기 때문에 ViewModel의 onCleared()가 호출되면(Activity 혹은 Fragment의 onDestroy 시점) 명시적으로 dispose해주어야 합니다.


### DisposableViewModel
위에서 설명했듯이 











안드로이드 앱을 개발할때 [Dagger](https://google.github.io/dagger/) 와 같은 [Dependency Injection](https://ko.wikipedia.org/wiki/%EC%9D%98%EC%A1%B4%EC%84%B1_%EC%A3%BC%EC%9E%85)(이하 DI)을 사용하는 것을 종종 볼 수 있습니다. 각 컴포넌트간의 의존성을 외부 컨테이너에서 관리하는 방식을 통해 코드 재사용성을 높이고 Unit Test도 편하게 할 수 있게 되는 장점을 가지고 있습니다. 그런데 최근에는 코틀린이 안드로이드 앱 개발 공식 언어가 되면서 코틀린의 다양한 언어적 장점을 이용한 DI 라이브러리가 생겨났고 그 중 하나인 [Koin](https://beta.insert-koin.io/) 라이브러리에 알아보려고 합니다. 
	
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



