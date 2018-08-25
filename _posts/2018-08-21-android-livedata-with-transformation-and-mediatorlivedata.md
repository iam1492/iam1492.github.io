---
title: Android LiveData, Tranformations 그리고 MediatorLiveData를 이용한 리액티브 앱 개발
description: 안드로이드 LiveData, Tranformations 그리고 MediatorLiveData를 이용한 리액티브 개발 방법 설명
layout: single
category: post
comments: true
author_profile: true
tags:
  - LiveData
  - Tranformations
  - MediatorLiveData
  - Android
  - MVVM
header:
  overlay_image: /assets/images/cover_img_livedata.jpg
  overlay_filter: 0.3 
  caption: "Photo credit: [**Pixabay**](https://pixabay.com/)"
---

이전 [포스팅]({% post_url 2018-08-13-android-mvvm-viewmodel-livedata-databinding %})에서는 MVVM 구조와 이를 도와주는 몇가지 컴포넌트에 대해서 정리를 했었는데 그 중에서 개인적으로 앞으로 활용도가 점점 많아 질 것으로 생각되는 LiveData에 대해서 조금더 자세히 알아보겠습니다. 그리고 간단한 [Sample Project](https://github.com/iam1492/LiveDataSample) 도 함께 만들어보겠습니다.

# LiveData

LiveData는 Activity, Fragment 와 같은 View와 View에게 보여질 데이터 사이의 안전한 결합을 제공하기 위해 탄생했습니다. Activity(또는 Fragment)의 라이프사이클을 알고 있기 때문에 Activity가 destroyed 상태에서 뷰를 갱신하려다가 앱이 죽는 문제가 생기지 않게 됩니다. 기존에 수많은 앱들이 이런 문제를 해결하기 위해 BaseActivity 클래스를 만들고 isActive 와 같은 flag를 두고 액티비티의 상태를 체크했다면 이제 LiveData를 통해 뷰를 갱신하면 더이상 이런 상태 체크를 하지 않아도 됩니다.

![safe_live_data](/assets/images/safe_live_data.png)
이와 같이 LiveData는 Observer 패턴을 이용하여 연결이 되기 때문에 View가 destroyed 되면 그와 동시에 연결또한 끊어지게 됩니다. 즉, LiveData에 observe 하는 주체(Activit, Fragment)가 사라지게 되는 것이죠.

LiveData는 ViewModel 내에서 주로 사용하게 됩니다. ViewModel안에서 다른 컴포넌트들과 연결된 상태로 다양한 이벤트를 받아 뷰를 갱신하게 됩니다. LiveData를 사용하면 다음과 같은 장점을 갖게 됩니다.

* Lifecycle을 알고 있어서 메모리릭이나 예기치 못한 crash를 방지할 수 있습니다.
* Data 자체의 변화를 observe하고 있기 때문에 Data가 항상 최신성을 유지하게 됩니다.

다만, LiveData 자체만으로는 RxJava처럼 강력한 스트림 기능이나 쓰레드 관리 기능이 없기 때문에 사용하는데 어느정도 제한적일 수 있습니다. 
구글에서는 이를 어느정도 보완할 수 있도록 [Transformations class](https://developer.android.com/reference/android/arch/lifecycle/Transformations)를 제공하고 있는데 이 클래스는 **map**, **switchMap** 함수를 통해 LiveData를 변환하거나 결합할 수 있는 기능을 제공합니다. 이 두 함수에 대해서는 곧 자세히 설명해보겠습니다.

### MutableLiveData

LiveData<T> 자체는 immutable(불변)하기 때문에 데이터를 갱신하지 못합니다. MutableLiveData는 LiveData를 상속받아 postValue와 setValue 함수를 제공하고 있으며 데이터를 갱신하고 싶을때 사용가능합니다. 만약 현재 쓰레드가 UI쓰레드라면 바로 setValue 함수를 사용할 수 있지만 백그라운드 쓰레드라면 postValue 함수를 사용해야합니다.  

### LifecycleOwner

LiveData를 observe 하는 주체는 반드시 [LifecycleOwner Interface](https://developer.android.com/reference/android/arch/lifecycle/LifecycleOwner)를 구현한 class 이어야 합니다. 대표적으로 Activity와 Fragment가 LifecycleOwner의 구현체 이며 내부적으로 lifecycle을 제공하기 위한 구현을 하고 있습니다. 만약 개발자가 직접 만든 커스텀 클래스에 LiveData를 observe 하고 싶다면 자체적으로 해당클래스에 LifeCycleOwner Interface를 구현해서 커스텀 클래스가 lifecycle을 알 수 있게 만들어줘야 합니다.

# Transformations

Transformations는 특정 LiveData의 값에 트리거가 일어나면(즉,값이 세팅이 되면) 그 트리거와 함께 항상 원하는 변환을 하여 새로운 LiveData를 만들고 싶을때 유용하게 사용할 수 있는 두 가지 함수를 제공합니다. 바로 map과 switchMap입니다. 사실 내부적으로는 두 함수 모두 [MediatorLiveData](https://developer.android.com/reference/android/arch/lifecycle/MediatorLiveData)라는 클래스를 활용하고 있기 때문에 원한다면 얼마든지 유사한 기능을 만들어서 사용할 수 있습니다.

### 1. Transformations.map

```java
public static <X, Y> LiveData<Y> map(
            @NonNull LiveData<X> source,
            @NonNull final Function<X, Y> mapFunction)
```

map 함수는 RxJava의 map과 비슷한 느낌입니다. 즉, LiveData에 특정 값이 세팅되어 트리거 되는 순간 전달된 Function() 함수를 통해 새로운 값을 가진 LiveData를 리턴합니다. map의 두번째 인자로 함수를 전달하는데 이 함수는 새로운 LiveData에 세팅될 value(Y 타입의 값) 자체를 리턴하게 됩니다. 코드로 이해하는게 가장 빠를 것 같습니다.

> ViewModel

```kotlin
private val _price = MutableLiveData<Int>()
val formattedPrice: LiveData<String> = Transformations.map(_price) {
    //새로운 Type의 값을 리턴. _price가 10000 이라면 "10,000" 이라는 String이 리턴
    DecimalFormat("#,###").format(it) 
}

fun setPrice(price: Int) {
    _price.value = price
}
```

> Activity

```kotlin
viewModel.setPrice(value)
viewModel.formattedPrice.observe(this, Observer {
	viewBinding.priceResult.text = it
})
```

위의 예제는 **Transformations.map**을 이용해 숫자를 가격 포맷으로 변경해주고 있습니다. 현재 구현상으론 액티비티에서 setPrice를 하고 있지만 이런 과정없이 api call을 통해 서버로부터 price값을 Int 값으로 받아온다면 위처럼 _price.value를 채워주는 순간 formattedPrice LiveData 역시 업데이트가 됩니다. 그럼 액티비티는 언제든 포맷화된 가격으로 UI에 표시를 해줄 수 있겠죠. 

아래는 kotlin extension으로 map 함수를 직접 구현한것입니다. extension을 활용해 직접구현해서 사용하면 좀더 직관적으로 사용할 수 있습니다.

```kotlin
inline fun <X,Y> LiveData<X>.map(crossinline block: (X) -> Y): LiveData<Y> {
    return MediatorLiveData<Y>().apply {
        addSource(this@map) {
            this.value = block.invoke(it)
        }
    }
}
//kotlin extension으로 조금더 직관적으로 사용
val formattedPrice: LiveData<String> = _price.map {
	DecimalFormat("#,###").format(it)
}
```

### 2. Transformations.switchMap

```java
public static <X, Y> LiveData<Y> switchMap(
            @NonNull LiveData<X> source,
            @NonNull final Function<X, LiveData<Y>> switchMapFunction)
```

switchMap 함수는 RxJava의 flatMap과 비슷한 느낌입니다. switchMap의 source로 등록된 LiveData에 값이 세팅되어 트리거가 일어나면 해당 값을 이용하여 다른 새로운 LiveData를 가져와 리턴합니다. 즉, map과의 차이점은 map은 값자체를 리턴하는 Function을 전달해야한다면 switchMap은 LiveData<Y> 를 리턴하는 Function을 전달해야합니다. 이러한 장점은 여러 LiveData를 겹합하는게 가능해집니다.
예를들면, 어떤 사용자의 전화번호가 변경이 되어서 phoneNumber LiveData가 트리거 될때 userLiveData = Transformations.switchMap(phoneNumber) {..}  이렇게 선언 되어있다면 switchMap 내부에서 phoneNumber를 이용해 Room db를 쿼리해 LiveData<User> 를 받아올 수 있습니다. 

```kotlin
private val phoneNumber = MutableLiveData<String>()
val user: LiveData<User> = Transformations.switchMap(phoneNumber) { phone ->
    getUserByPhone(phone) //LiveData<User> 를 리턴
}
```

사용방법은 간단합니다. getUserByPhone은 실제 서비스에서는 아마 DB나 Network으로 부터 반환받은 값이 될 수 있을 것 같습니다.

### Transformations 사용시 주의할 점

Transformations은 코드의 initialize 시점에 사용해야 원하는 동작을 얻을 수 있습니다.
아래는 switchMap을 사용하는 잘못된 예를 보여주고 있습니다. 코드의 출처는 구글 개발자의 [블로그 포스팅](https://medium.com/androiddevelopers/livedata-beyond-the-viewmodel-reactive-patterns-using-transformations-and-mediatorlivedata-fda520ba00b7)에서 가져왔습니다.

```kotlin
var lateinit randomNumber: LiveData<Int>
//버튼 클릭할때 불리우는 함수
fun onGetNumber() {
   randomNumber = Transformations.map(numberGenerator.getNumber()) {
       it
   }
}
```

위에서는 switchMap을 사용할때 특정 이벤트가 일어날때마다 Transformations.map을 반복 수행하게 되어있습니다. 이렇게 되면 randomNumber 는 다시 새로운 객체로 대체되는데 이 객체는 Activity에서 observe한 객체와는 다른 객체이기 때문에 Activity에서 이벤트를 받지 못하게 됩니다. 

## MediatorLiveData

![mediator_live_data](/assets/images/mediator_live_data.png)

MediatorLiveData는 클래스 이름에서 유추가 가능하듯 여러개의 LiveData를 Mediate(중재) 하는 역할을 합니다. 사실 실제 기능 자체는 단순한데 addSource() 함수와 removeSource() 함수를 통해 여러개의 LiveData를 하나의 MediatorLiveData의 source로 등록/해제가 가능합니다. 이렇게 등록을 하면 source로 등록된 LiveData의 값이 변경될때마다 onChanged 이벤트를 통해 변경된 값을 전달 받게 됩니다.

### MediatorLiveData 활용 - 더하기 예제

이해를 돕기 위해 간단한 더하기 예제를 만들어보겠습니다. 숫자 2개를 입력하면 입력하는 동시에 더하기가 계산되어 결과를 보여주는 아주 간단한 예제 입니다. 먼저 X + Y = Z 이라고 하면 X과 Y에 해당하는 좌항 우항을 MutableLiveData<Int> 로 만들고 그 결과인 Z를 MediatorLiveData<Int> 로 만들어보겠습니다.

```kotlin
class TestViewModel: ViewModel() {

    private val leftOperand = MutableLiveData<Int>()
    private val rightOperand = MutableLiveData<Int>()

    private val _plusMediator = MediatorLiveData<Int>()
    val plusMediator: LiveData<Int>
        get() = _plusMediator

    ...
}
```

위와 같이 두개의 좌항/우항은 MutableLiveData<Int>로 그리고 덧셈의 결과를 저장할 MediatorLiveData<Int> 객체를 하나 만들었습니다. 

이전 포스팅인 [Android MVVM 패턴, ViewModel, LiveData, Databinding을 이용해 간단한 Toy App 만들기]({% post_url 2018-08-13-android-mvvm-viewmodel-livedata-databinding %}) 에서도 이야기 했듯이 외부에서 값을 변경하면 안되는 경우 변경가능한 _plusMediator(Mutable)를 private로 선언하고 plusMediator 프로퍼티를 새로 만들어 _plusMediator를 LiveData<Int>(Immutable) 형태로 외부로 노출해야 안전합니다.

이제 MediatorLiveData에 두 LiveData를 addSource() 함수를 이용해 observe해보겠습니다.

```kotlin
class TestViewModel: ViewModel() {
	init {
	    _plusMediator.addSource(leftOperand) {
	    	_plusMediator.value = plusOperands()
	    }
	    _plusMediator.addSource(rightOperand) {
	        _plusMediator.value = plusOperands()
	    }
	}

	private fun plusOperands(): Int 
		= (leftOperand.value ?: 0) + (rightOperand.value ?: 0)
	fun setLeftOperand(leftValue: Int) {
		leftOperand.value = leftValue
	}
	fun setRightOperand(rightValue: Int) {
		rightOperand.value = rightValue
    	}
}
```

두개의 LiveData(leftOperand, rightOperand)를 source로 추가했습니다. 이렇게 하면 이제 각각의 LiveData의 값이 변경됐을때 onChanged(위 예제에서는 람다식으로 대체) 이벤트를 받게 됩니다. plusOperands() 함수는 단순히 두개의 LiveData의 값을 더해서 리턴해줍니다. MediatorLiveData는 source를 observe 하기도 하지만 자기자신도 LiveData이기 때문에 당연히 값을 가질 수가 있습니다. setLeftOperand() 와 setRightOperand() 는 View(Activity)에서 각각이 좌항/우항에 UI로부터 입력받은 값을 넣기 위해 사용합니다. 각각의 값이 넣어질때마다 _plusMediator에도 값이 세팅될것입니다.

자 이제 ViewModel에서 데이터는 준비됐으니 View(Activity)에서 어떻게 접근하는지 살펴보겠습니다.

```kotlin
class MainActivity : AppCompatActivity() {

    lateinit var viewBinding: ActivityMainBinding

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)

        viewBinding = DataBindingUtil.setContentView(this, R.layout.activity_main)
        val viewModel = ViewModelProviders.of(this).get(TestViewModel::class.java)

        viewBinding.leftOperand.afterTextChanged {
            val value = if (it?.toString()?.isNotBlank()  == true) it.toString().toInt() else 0
            viewModel.setLeftOperand(value)
        }

        viewBinding.rightOperand.afterTextChanged {
            val value = if (it?.toString()?.isNotBlank()  == true) it.toString().toInt() else 0
            viewModel.setRightOperand(value)
        }

        viewModel.plusMediator.observe(this, Observer {
            viewBinding.plusResult.text = it.toString()
        })
    }
}
```
viewBinding.leftOperand 와 viewBinding.rightOperand 는 안드로이드 EditText 입니다. 각각의 입력창은 텍스트가 입력될때마다 viewModel의 LiveData에 값을 세팅해줍니다 그리고 viewModel.plusMediator를 Observe 하면 덧셈연산이 완료되면 더해진 값을 전달받고 UI에 표시를 하게 됩니다.

참고로 여기서 사용되고 있는 afterTextChanged는 코틀린 익스텐션을 이용해서 addTextChangedListener를 조금 편하게 사용할 수 있게 구현을 한것입니다.

```kotlin
inline fun EditText.afterTextChanged(crossinline block: (editable: Editable?) -> Unit) {
    this.addTextChangedListener(object : TextWatcher {
        override fun afterTextChanged(p0: Editable?) {
            block.invoke(p0)
        }
        override fun beforeTextChanged(p0: CharSequence?, p1: Int, p2: Int, p3: Int) {}
        override fun onTextChanged(p0: CharSequence?, p1: Int, p2: Int, p3: Int) {}
    })
}
```

### MediatorLiveData 활용 - 구글 IO 2018 코드중에

MediatorLiveData는 다양하게 활용할 수 있는데 [구글io 소스 코드](https://github.com/google/iosched)에서 활용하고 있는 부분을 잠깐 보고 넘어가겠습니다.
> ScheduleViewModel.kt

```kotlin

//에러처리
_errorMessage.addSource(loadSessionsResult) { result ->
    if (result is Result.Error) {
        _errorMessage.value = Event(content = result.exception.message ?: "Error")
    }
}

//성공처리
_sessionTimeDataDay1.addSource(loadSessionsResult, {
    //요청이 성공한 경우만 아래 실행 아니면 리턴 [코드 생략]
    _sessionTimeDataDay1.value = _sessionTimeDataDay1.value?.apply {
        list = userSessions
    } ?: SessionTimeData(list = userSessions)
})
```

소스를 보다보면 위와 같은 코드들을 많이 볼 수 있습니다. 눈치로 대충 어떤일을 하는지 감이 오게되는데 loadSessionResult는 사용자 세션을 얻기 위한 요청의 LiveData 형태의 응답입니다. 이때 loadSessionResult 이벤트가 트리거 되면 두 MediatorLiveData인 _errorMessage와 _sessionTimeDataDay1 의 onChanged가 불리게 되고 위의 코드에서는 error일때와 success 일때를 서로 다르게 처리하고 있습니다. 이렇게 같은 요청이지만 요청한 결과 따라 View에는 필요한 이벤트만 전달이 가능하게 되고 코드 역시 잘 분리가 됩니다.

LiveData 자체로만은 실제 프로덕션 앱에서 다양한 케이스를 구현하기 불편한 부분이 있지만 Transformations class 와 MediatorLiveData를 함께 잘 활용해 준다면 MVVM 구조를 유지하면서도 복잡한 케이스도 어느정도 어렵지 않게 잘 구현 할 수 있지 않을까 기대합니다. 

Sample 프로젝트는 github에 [LiveDataSample](https://github.com/iam1492/LiveDataSample)에 올려두었습니다.
 



