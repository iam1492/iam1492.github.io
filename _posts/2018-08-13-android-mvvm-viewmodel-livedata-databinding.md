---
title: Android MVVM 패턴, ViewModel, LiveData, Databinding을 이용해 간단한 Toy App 만들기
layout: single
category: post
comments: true
author_profile: true
tags:
  - mvvm
  - viewmodel
  - livedata
  - android
  - databinding
---

MVVM은 Model - View - ViewModel의 줄임말로 최근 안드로이드 아키텍쳐에 가장 많은 사랑을 받으며 구글의 전폭적인 지원을 받기도 하고 있습니다. 

[당근마켓](https://www.daangn.com/)의 경우에는 2015년에 처음 개발을 시작했고 당시에 트렌드였던 MVP로 구조로 개발이 되어 있습니다. 점점 서비스 규모가 커지면서 혼자 개발함에도 불구하고 관리의 어려움이 많았고 테스트 코드에 대한 니즈가 생기면서 구조적인 변화를 고민하게 되었습니다. 그래서 최근에 Clean Architecture, MVVM 등 Architecture에 관심이 생겼고 그 중 MVVM 관련해서 정리를 해보려고 합니다. MVVM은 말그대로 아키텍쳐 패턴입니다. MVVM이 어떻게 탄생했는지에 대해서는 [위키피디아](https://en.wikipedia.org/wiki/Model%E2%80%93view%E2%80%93viewmodel)를 참고해보세요 :) 개념설명과 함께 Naver open api를 이용한 [단축 url 생성 Sample App](https://github.com/iam1492/mvvmsample)도 함께 만들어보겠습니다.

> 이 포스팅에서의 View는 안드로이드의 UI컴포넌트 기본단위 View 가 아닌 MVVM 에서의 View를 이야기 합니다.

## MVVM 구조의 장점

MVVM 패턴을 지켜 개발된 앱은 아래와 같은 특징을 갖게 됩니다.
1. [Separation of concern](https://en.wikipedia.org/wiki/Separation_of_concerns) - 하나의 소프트웨어를 최대한 기능적으로 작은 단위로 나눔 
2. 테스트가 쉬워지고 큰프로젝트도 상대적으로 관리하기 좋다.
3. [SOLID principle](https://en.wikipedia.org/wiki/SOLID) 을 지향 
4. 앱이 구조적으로 약한 결합의 컴포넌트로 나눠짐

대체적으로 위와같은 장점에 대해서 이야기를 하지만 결국 가장 큰 목적은 유지보수가 쉽고 테스트가 용이한 코드를 만드는 것입니다.

![mvvm](/assets/images/mvvm.png){: .align-center}

MVVM의 기본 구조를 그림으로 표현한 것입니다. View는 ViewModel에게 클릭이벤트, 필요한 데이터 요청등을 명시적으로 하고 ViewModel이 nofity할때까지 기다리게 됩니다. 그와 동일하게 ViewModel은 Model을 통해 데이터를 요청하고 기다리게 됩니다. 각각의 컴포넌트간 레퍼런스를 갖지않고 단방향(View -> ViewModel -> Model)의 디펜던시만을 갖게 됩니다.

## Android Architecture Component

구글에서 제공하는 [Android Architecture Component](https://developer.android.com/topic/libraries/architecture/)(이하 AAC)에서 제공하는 라이브러리를 이용하면 MVVM 구조로 앱 개발을 더 쉽게 할 수 있습니다. MVVM 패턴 이라고 말하면 디자인 패턴중 하나이기 때문에 반드시 AAC 라이브러리를 사용할 필요는 없습니다. 하지만 AAC 라이브러리에는 ViewModel 추상클래스를 명시적으로 제공하며 이 추상클래스를 통해 **lifecycle aware viewmodel**를 만들 수 있게 도와줍니다.
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
1. DataModel 이라고도 하며 Network, DB, SharedPreference등 다양한 데이터소스로 부터 필요한 데이터를 준비합니다.
2. ViewModel 에서 데이터를 가져갈 수 있게 데이터를 준비하고 그에 대한 "이벤트"를 보냅니다. 이번 예제에서는 이와 같은 Observe Pattern을 RxJava를 이용해 구현했습니다.

## LiveData
예제 코드를 알아보기 전에 마지막으로 AAC에 포함된 [LiveData](https://developer.android.com/topic/libraries/architecture/livedata)에 대해 간단히 살펴보겠습니다. LiveData는 "observable data holder class"로 특정 데이터를 옵져버블하게 만들게 됩니다. RxJava의 Observable과의 가장 큰 차이점은 LiveData의 경우 View lifecycle에 따라 필요한 일을 알아서 해주기 때문에 개발자가 따로 신경을 쓰지않아도 됩니다. MVVM에서는 ViewModel 내부에서 데이터를 외부로 노출시키려는 목적으로 LiveData를 사용합니다.
LiveData는 아래와 같은 특징을 갖습니다.
1. **Lifecycle aware simple observer** - 여기서 simple인 이유는 단순히 하나의 데이터에 observe를 할 수 있게만 해주기 때문.
2. **Activity, Fragment is LifecycleOwner** - LiveData와 바인딩을 하기 위해서는 LifecycleOwner interface 구현체여야 합니다. 실제 최근 안드로이드 소스코드를 보면 AppCompatActivity, Fragment 모두 LifecycleOwner를 구현하고 있습니다.
3. **Lifecycle 을 알기 때문에 View가 destroy되면 알아서 Observe상태를 해제합니다.** (rxjava의 dispose를 lifecycle에 따라 자동으로 수행) 이런 특징은 다음의 장점이 있습니다.
    - lifecycle에 따라 알아서 observing을 해제하여 메모리릭이 안생김
    - 백그라운드에서 View를 건드려 앱이 죽는 문제가 사라짐.
4. **View와 ViewModel의 LiveData간의 디펜던시가 단방향으로 약한 결합을 가지게 됩니다.**
5. **만약 LiveData 대신 Rxjava를 쓴다면?** 당연히 개발자가 lifecycle에 맞춰 dispose를 해줘야할 책임이 생기게 됩니다.


# Toy App 구현해보기 - 단축 Url 생성앱

![shortenurlapp](/assets/images/shorten_url_app.gif){: .align-center}

이제 위의 개념이 어느정도 감이 왔다면 코드를 보면서 실제 어떻게 구현이 되는지 알아보겠습니다. 샘플로 만들 앱은 사용자가 특정 URL을 입력하면 naver api를 이용해 단축 URL을 만들어주는 단순한 앱입니다. 먼저 샘플앱에서 사용할 라이브러리를 간단히 소개해볼게요
1. **Koin** - [Dependency Injection](https://ko.wikipedia.org/)을 위해 사용합니다. 
2. **Retrofit** - Naver api 를 호출하기 위한 REST Api 라이브러리 입니다.
3. **RxJava** - Model에서 데이터를 노출하고 이벤트를 발생시키기 위해 사용합니다. Model에 observe 하는 주체는 ViewModel이 되겠죠.
4. **LiveData** - ViewModel에서 View에 데이터를 노출하고 이벤트를 발생시키기 위해 사용합니다. 여기선 View가 ViewModel에 observe를 하는 주체가 됩니다.
5. **Databinding** - ViewModel과 xml layout간의 데이터를 바인딩하기 위해 사용합니다.

이 포스팅에서는 Dependency Injection에 구현방법에 대해서는 생략하겠습니다. Koin 라이브러리의 사용법이 궁금하다면 이전 포스팅인 [Kotlin 안드로이드 의존성 주입 라이브러리 Koin을 소개합니다.]({% post_url 2018-07-22-dependency-injection-with-kotlin-koin %}) 를 참고해주세요.

## Package 구조

<img src="/assets/images/packages.png" width="460" class="align-center">

패키지 구조는 이해를 돕기 위해 의도적으로 view, viewmodel, model로 나누었습니다. 
각각의 구현부분은 최대한 단순하게 만들려고 노력했습니다.

![mvvmimpl](/assets/images/mvvm_impl.png){: .align-center}

실제 구현은 위의 그림과 같습니다. xml과 ViewModel과의 바인딩을 위한 DataBinding library를 사용하였고 ViewModel은 LiveData를 이용하여 데이터를 노출시키고 Model은 RxJava를 이용하여 데이터를 노출하게 됩니다.

## Model
모델은 네트워크를 통해 naver shorten url API를 요청해 데이터를 가져오는 역할을 합니다.

```kotlin
import io.reactivex.Single

interface Repository {
    fun getShortenUrl(url: String): Single<ShortenUrl>
}
```

```kotlin
import com.daangn.www.mvvmsample.api.Api
import io.reactivex.Single

class NetworkRepositoryImpl(private val api: Api): Repository {
    override fun getShortenUrl(url: String): Single<ShortenUrl> {
        return api.shorturl(url)
            .map { shortenUrlResponse ->
                shortenUrlResponse.result
            }
    }
}
```

실제 코드 구현은 매우 간단하게 되어있습니다. 여기서 ShortenUrl은 단축url 데이터를 담을 kotlin data class 입니다.
Api response spec상 몇가지 메타 데이터를 함께 내려주고 있어서 result 값인 ShortenUrl만 넘겨주고 있습니다. 만약 실제 프로덕션 앱이라면 메타데이터를 함께 처리하기 위한 로직이 필요할 것 같습니다.
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

DisposableViewModel은 AAC의 ViewModel.class 의 구현체이며 ShortenUrlViewModel의 상위 클래스입니다. ViewModel은 Model(여기에선 Repository)이 제공하는 데이터에 subscribe를 하게 되는데 ViewModel이 클리어되는 시점에 RxJava의 disposable또한 함께 잘 클리어되는 것을 보장해 주기 위한 베이스 클래스라고 보면 됩니다. 즉, RxJava의 경우 LiveData와는 다르게 lifecycle을 알지 못하기 때문에 ViewModel의 onCleared()가 호출되면(Activity 혹은 Fragment의 onDestroy 시점) 명시적으로 dispose해주어야 메모리릭을 방지할 수 있습니다.

```kotlin
class ShortenUrlViewModel(private val repository: Repository) : DisposableViewModel() {

    private val _shortenUrl = MutableLiveData<String>()
    private val _error = MutableLiveData<String>()
    private val _clickCopyToClipboard = SingleLiveEvent<String>()
    ...

    val showResult = MutableLiveData<Boolean>()

    //mutableLiveData를 immutable 하게 노출
    val shortenUrl: LiveData<String> get() = _shortenUrl
    val error: LiveData<String> get() = _error
    val clickCopyToClipboard: LiveData<String> get() = _clickCopyToClipboard
    ...

    fun getUrlValidator(errorMessage: String): METValidator {
        return RegexpValidator(errorMessage, Patterns.WEB_URL.pattern())
    }

    fun getShortenUrl(url: String) {
        addDisposable(repository.getShortenUrl(url)
            .subscribeOn(Schedulers.io())
            .observeOn(AndroidSchedulers.mainThread())
            .subscribe({
                showResult.postValue(true)
                _shortenUrl.postValue(it.url)
            }, {
                _error.postValue(it.message)
            }))
    }

    fun clickConvert() {
        _clickConvert.call()
    }

    ...
}
```

ShortenUrlViewModel은 DisposableViewModel 상속받습니다. ViewModel에서는 UI에 노출할 데이터나 이벤트를 MutableLiveData<T>를 이용해 구현하고 있습니다.
```kotlin
private val _shortenUrl = MutableLiveData<String>() 
val shortenUrl: LiveData<String> get() = _shortenUrl
```
LiveData 프로퍼티를 위와 같이 노출 하는 것은 ViewModel 내부에서는 Mutable 한 데이터를 외부에서는 Immutable 하게 사용하도록 제약을 주기 위해서입니다. 즉, 이렇게 하면 View에서 ViewModel의 데이터 상태를 변경하지 못하게 됩니다.
이런식의 코드 컨벤션은 코틀린 [공식 다큐먼트](https://kotlinlang.org/docs/reference/coding-conventions.html#property-names)를 참고했습니다.

View에서 getShortenUrl() 함수를 호출하면 ViewModel은 Model의 Repository로 부터 shortenUrl을 가져오게되고 가져온 데이터가 준비되면 _shortenUrl.postValue(url) 을 통해 observe 하고 있던 View에 노티를 주게 됩니다.

위의 코드를 보면 SingleLiveEvent라는 특이한 놈이 있는데 SingleLiveEvent는 MutableLiveData를 상속받은 클래스로 ViewModel에서 일회성 이벤트를 처리하기 위한 일종의 편법을 구글개발자중 한명이 구현한것입니다.
사실 클릭 이벤트 하나만을 처리하기 위한 부분으로 상당히 오버하는 느낌이지만 개념자체는 이해해두면 LiveData를 더 잘 이해하는데 도움이 됩니다.
LiveData는 그 특성상 lifecycle에 따라 안드로이드 디바이스의 화면전환이 되면 데이터를 보존하기 위해 observe 하고 있던 View(LifecycleOwner)에게 다시 notification을 줍니다. 
하지만 클릭 이벤트의 경우 이런 lifecycle과 상관없이 사용자의 액션에 반응하고 그 이외의 경우에는 반응을 하면 안되기 때문에 이런 편법을 사용하게 되는것입니다.

SingleLiveEvent를 더 자세히 알고 싶다면 이 문제에 대한 [블로그 포스팅](https://medium.com/google-developers/livedata-with-snackbar-navigation-and-other-events-the-singleliveevent-case-ac2622673150)을 한번 읽어보세요 :)

## View (Activity)

```kotlin
class ShortenUrlActivity : BaseActivity<ActivityShortenUrlBinding>() {

    override val layoutResourceId: Int = R.layout.activity_shorten_url

    private val shortenUrlViewModelFactory: ShortenUrlViewModelFactory by inject()

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)

        val shortenUrlViewModel = ViewModelProviders.of(this, shortenUrlViewModelFactory).get(ShortenUrlViewModel::class.java)

        shortenUrlViewModel.clickConvert.observe(this, Observer {
            shortenUrlViewModel.getShortenUrl(viewDataBinding.urlEditText.text.toString())
        })

        shortenUrlViewModel.error.observe(this, Observer<String> { t ->
            Toast.makeText(this@ShortenUrlActivity, t, Toast.LENGTH_LONG).show()
        })

        ...
        
        viewDataBinding.shortenUrlViewModel = shortenUrlViewModel
        viewDataBinding.setLifecycleOwner(this)
    }
}
```

ShortenUrlActivity 에서는 ViewModelProviders를 통해 ShortenUrlViewModel 클래스를 생성합니다. 이때 ShortenUrlViewModel이 디폴트 생성자만 사용한다면 따로 [ShortenUrlViewModelFactory](https://github.com/iam1492/mvvmsample/blob/master/app/src/main/java/com/daangn/www/mvvmsample/viewmodel/ShortenUrlViewModelFactory.kt) 와 같은 팩토리 클래스를 만들필요가 없습니다. 하지만 이 예제의 경우 ShortenUrlViewModel이 생성자로 Repository를 받고 있기 때문에 따로 ViewModelFactory를 만들어 ViewModel을 생성해야합니다.

View(ShortenUrlActivity)에서는 생성한 ViewModel의 레퍼런스를 이용해 클릭이벤트(clickConvert) 혹은 error state 등을 observe 하고 있는걸 볼 수 있습니다. 그런데 getShortenUrl() 요청의 결과값은 왜 observe하고 있지 않을까요?

```kotlin
viewDataBinding.shortenUrlViewModel = shortenUrlViewModel
viewDataBinding.setLifecycleOwner(this)
```

코드 맨 아래 부분을 보면 Android Databinding을 통해 xml layout에 ViewModel을 바인딩 해주고 있습니다. 그리고 setLifecycleOwner() 함수를 이용해 lifecycleOwner(예제에서는 ShortenUrlActivity)만 설정을 해주면 ViewModel에서 postValue를 해주는 순간 UI가 업데이트 되기 때문에 따로 observe를 할 필요가 없습니다.

아래는 ShortenUrlActivity 의 xml 레이아웃 파일입니다.

> activity_shorten_url.xml

```xml
<data>
    <variable
        name="shortenUrlViewModel"
        type="com.daangn.www.mvvmsample.viewmodel.ShortenUrlViewModel" />
</data>

<TextView
    android:id="@+id/result_url"
    android:layout_width="0dp"
    android:layout_height="wrap_content"
    android:text="@{shortenUrlViewModel.shortenUrl}"
    .. />

<Button
    android:id="@+id/button"
    android:layout_width="0dp"
    android:layout_height="wrap_content"
    android:text="@string/convert"
    android:onClick="@{() -> shortenUrlViewModel.clickConvert()}"
    ../>
```

layout xml 에서는 실제 shortenUrlViewModel 객체를 이용해 이벤트를 호출하거나 ViewModel에 정의된 LiveData를 바인딩하고 있습니다.

LiveData 나 DataBinding등 각각의 컴포넌트들도 더 많은 기능들이 있지만 이번 포스팅은 이정도로 마칠까 합니다. 아직은 클릭 이벤트 처리라던가 각기 다른 도메인 데이터를 매핑한다던가 하는 부분에 있어서 과도기적인 부분이 분명 있는것 같지만 구글의 아키텍쳐에 대한 열망이 식지 않는한 계속해서 발전하지 않을까 하는 생각이 듭니다. :blush:

Toy App 프로젝트는 github에 [MVVMSample](https://github.com/iam1492/mvvmsample)에 올려두었습니다.



