---
title: 안드로이드 Dagger로 의존성 주입 조금더 잘 사용해보기
description: 안드로이드 Dagger 라이브러리 다시 제대로 배워보기
layout: single
category: post
comments: true
author_profile: true
tags:
  - Dagger 
  - Android
  - DI
  - Dependency Injection
header:
  overlay_image: /assets/images/android_dagger_rethink.jpg
  overlay_filter: 0.3 
  caption: "Photo credit: [**Pixabay**](https://pixabay.com/)"
---

[Dagger 2](https://google.github.io/dagger/)는 현재 안드로이드 의존성 주입(Dependency Injection) 라이브러리로 가장 많이 사용되고 있습니다. 의존성 주입을 위한 강력한 기능을 많이 지원하고 있지만 가장 큰 단점은 배우기 어렵다는 점입니다. 

즉 러닝커브(learning curve) 가 높은 편이어서 많이 사용되고 있는 만큼 잘못 사용되기도 쉽습니다. 또 많은 블로그에서 Dagger를 다루고 있지만 그만큼 서로 다른 방법으로 설명을 하고 있고 어떤 방식으로 개발에 적용하는게 좋은지 판단하기 어려운 경우도 많이 있습니다. 그리고 어노테이션 기반이라 컴파일 시간이 늘어난다는 단점도 있는 반면 컴파일 타임에 미리 에러를 발생시킨다는 큰 장점도 있습니다. 최근에는 코틀린으로 바뀌면서 코틀린언어로 개발된 여러 대체 라이브러리가 생기고 있기도 하지만 여전히 대다수의 서비스는 Dagger를 사용하고 있습니다. 그만큼 강력한 기능을 많이 가지고 있기도 합니다.

이번 포스팅에서는 Dagger 의 기본적인 사용법에 대해 설명하지는 않을 예정입니다. 아직 Dagger 를 한번도 사용해보지 않았다면 기본 문서나 Jake Wharton 의 [Dependency Injection with Dagger 2](https://jakewharton.com/dependency-injection-with-dagger-2/) 영상을 먼저 볼것을 추천드립니다. 대신 Dagger 를 사용할때 어떻게 하면 조금더 잘 사용할 수 있을지 혹은 큰 틀에서 어떻게 내 안드로이드 앱에 적용하면 좋을지에 대해 이야기해보려고 합니다.

먼저 일반적인 Dagger 라이브러리의 사용해 구성되는 앱의 모습을 보겠습니다.

## SubComponent를 활용한 구성

![dagger_before_android](/assets/images/dagger_before_android.png)

처음 Dagger 가 나오면서 많은 개발자들이 이를 서비스에 적용하기 시작합니다. 위의 그림은 Dagger의 초기 버전을 안드로이드 앱에 적용한 모습을 최대한 단순화 시켜본 것입니다.
앱내에 글로벌하게 사용할 수 있는 의존성 주입을 위해 Application Component와 Application Module을 두고 해당 Module에 Preference 객체 HttpClient 객체등을 주입 할 수 있게 준비하게 됩니다. 그리고 각각의 액티비티에 필요한 객체를 주입하기 위해 Activity SubComponent 와 해당 SubComponent에서 사용할 Module들을 정의합니다. 이러한 구성의 가장 큰 단점은 매번 액티비티 혹은 프래그먼트를 추가할 때마다 Subcomponent를 추가하는 불편함이 있다는 것입니다. 

## Dagger-Android
![dagger_after_android](/assets/images/dagger_after_android.png)

Dagger 라이브러리는 2.10버전 부터 안드로이드에서 좀더 편하게 사용할 수 있도록 [dagger-android, dagger-android-support 라이브러리](https://google.github.io/dagger/android.html)등을 추가로 제공합니다.
위의 그림에서 보듯이 새로 소개된 어노테이션을(@ContributesAndroidInjector) 이용해 Subcomponent가 자동으로 생성되므로 더이상 개발자가 직접 만들어줄 필요가 없어졌습니다. 개발자는 이제 각각의 액티비티나 프래그먼트에서 주입할 객체가 정의된 Module만 잘 정의해주면 됩니다.

## Dagger2 조금더 잘 사용해보기

Dagger 라이브러리를 사용하는 목적은 결국 사용하고 싶은 객체의 생성을 외부에 맡기고 외부에서 생성된 객체를 내가 사용하고자 하는 곳에 적절히 주입하는 것입니다. 라이브러리에서는 Module을 통해 객체를 제공하는데 이때 사용되는 어노테이션이 @Provides 입니다. 하지만 @Provides 외에도 몇가지 객체를 제공하기 위한 방법을 제공하고 있습니다. 바로 Constructor injection과(생성자에 @Inject 어노테이션 사용) @Binds 그리고 @IntoMap 또는 @IntoSet을 이용한 Multibinding 입니다.
그럼 언제 어떤방법을 사용하면 좋은걸까요? 대부분 블로그 포스팅에서는 @Provides를 많이 사용합니다. 하지만 @Provides를 사용하려면 항상 Module을 따로 정의해야하는 불편함이 있습니다. 그럼 언제 어떤 방법으로 제공될 객체를 정의하면 좋을지 한번 살펴보겠습니다.

> 1. Constructor injection 이 가능하면 우선적으로 고려한다.
> 2. 인터페이스(또는 추상클래스)를 사용하는 경우 @Provides 대신 @Binds를 사용할 것을 고려한다.
> 3. 1,2번의 경우가 다 해당안되면 @Provides를 사용한다.

저는 위의 순서로 DI를 적용하는것을 추천드립니다. 최대한 불필요한 코드를 줄이고 Dagger로 하여금 코드를 더 효율적으로 생산하도록 해주기 때문입니다.

## Constructor injection
생성자코드 앞에 @Inject 어노테이션을 사용하여 객체 주입이 가능한 경우 따로 @Provides를 사용할 필요가 없습니다. Constructor injection 은 주입하고자 하는 객체 클래스의 생서자가 내 프로젝트내에 있어서 접근/수정이 가능하면 사용이 가능합니다. 반대로 말하면 Calendar.getInstance(), Builder 패턴을 이용하는 클래스 또는 외부 라이브러리에서 제공되는 클래스라 <u>개발자가 생성자를 수정하지 못할때</u> Constructor Injection을 사용할 수 없다는 의미가 되기도 합니다.

```kotlin
//Wheel 객체와 Engine 객체 주입
class Car @Inject constructor(wheel: Wheel, engine: Engine) {
    val name = "car has ${wheel.name} and ${engine.name}"
}
class Wheel @Inject constructor(){
    val name = "Super Wheel"
}
class Engine @Inject constructor(){
    val name = "Super Engine"
}
```

위의 코드 처럼 @Inject 어노테이션을 생성자에 붙이는 것만으로 의존성 주입이 가능합니다. Car 객체 역시 다음처럼 주입이 가능하게 됩니다.
```kotlin
@Inject lateinit var car: Car //Car 객체 주입 
```

만약 Car class 에 새로운 객체를 주입하고 싶지만 그 객체가 Constructor injection으로 객체 주입이 어려운 경우라면, @Provides를 통해 모듈에 정의해야합니다. 

```kotlin
class Car @Inject constructor(wheel: Wheel, engine: Engine, chair: Chair) {
    val name = "car has ${wheel.name} and ${engine.name} and ${chair.name}"
}

//Chair는 빌더 패턴으로만 객체를 만들 수 있다고 가정합니다.
class Chair private constructor(val name: String?) {
    data class Builder(private var name: String? = null) {
        fun name(name: String) = apply { this.name =  name }
        fun build() = Chair(name)
    }
}

//때문에 Constructor injection는 사용불가하며 @Provides를 사용해야 합니다.
@Module
internal class CarModule {

    @Provides
    internal fun providesChair(): Chair = Chair.Builder()
            .name("Super Chair")
            .build()
}
```
Chair의 경우 빌더를 통해서만 객체 생성이 가능하고 private construtor 이기 때문에 Constructor Injection을 사용할 수 없기 때문에 모듈을 만들고 Chair 객체를 @Provides 해주어야합니다.

## Binds

Dagger 홈페이지의 [Faq섹션](https://google.github.io/dagger/faq#why-is-binds-different-from-provides)을 보면 Binds와 Provides의 차이점을 설명하고 있습니다. @Binds는 특정 인터페이스 클래스(혹은 추상클래스)의 구현체를 Provides하고자 할때 유용합니다. 즉, 특정 인터페이스의 구현체에 대한 객체생성의 경우 따로 객체 생성코드가 없어도 Dagger에서 추론하는 정보로 객체 생성이 가능하기 때문에 개발자가 @Provides를 통해 객체를 생성해줄 필요가 없습니다. @Binds 의 경우 하나의 파마메터만 받는 추상 메소드를 정의해야하고 리턴타입은 반드시 그 하나의 파라메터의 상위 인터페이스(또는 추상클래스)가 되어야 합니다. 이 부분은 코드를 보며 다시 설명해보겠습니다.

```kotlin
@Module
internal abstract class CarModule {

    @Binds
    abstract fun carPresenter(presenterImpl: CarPresenterImpl): CarPresenter
}

class CarPresenterImpl @Inject constructor(): CarPresenter {
    ...
}
```
위와 같이 단순한 CarPresenter 코드를 만들고자 할때 위처럼 @Binds 어노테이션과 함께 CarPresenter 를 리턴하고 구현체인 CarPresenterImpl 클래스를 1개의 파라메터를 받는 추상 메소드를 정의하면 됩니다. 그리고 CarPresenterImpl 구현부에서는 생성자에 @Inject 어노테이션을 추가해 줍니다. 
그러면 마찬가지로 뷰에서 CarPresenter를 주입할 수 있게 됩니다. 코드를 보면 알겠지만 추상메소를 정의해야하기 때문에 모듈역시 추상클래스여야 하는 제약이 있습니다. 한가지 단점이 있다면 @Binds와 @Provides메소드를 같은 모듈에 정의하지 못합니다. 만약 같은 모듈에 정의하고 싶다면 @Provides메소드를 **static 메소드**로 만들어줘야합니다.

```kotlin
@Module
internal abstract class CarModule {

    @Binds
    abstract fun carPresenter(presenterImpl: CarPresenterImpl): CarPresenter

    @Module
    companion object {
        @JvmStatic @Provides
        internal fun providesChair(): Chair = Chair.Builder()
                .name("Super Chair")
                .build()
    }
}
```
위에서 보면 Chair객체를 제공하기 위해 providesChair() 메소드를 static 메소드로 제공하고 있습니다. 
Google의 설명에 따르면 @Binds를 사용하는 경우 조금더 효율적인 코드가 generate 된다고 합니다. 또 실제 아주 약간이지만 개발자가 직접 객체를 생성할 필요가 없어서 코드량이 줄어들게 되는 장점도 있습니다. 

## 보너스 - MultiBinding (@IntoMap)
최근 공개된 [구글 IO 소스](https://github.com/google/iosched)를 보면 @IntoMap 과 @Binds를 이용해 ViewModel 및 ViewModelFactory의 객체 생성을 잘 관리하고 있는걸 볼수있습니다. @IntoMap을 이용한 Multi Binding은 하나의 객체를 주입하는게 아닌 객체를 담고 있는 컬렉션 자체를 주입하는 방법을 이야기 합니다. 여기서는 @IntoMap을 활용한 아주 간단한 예제 코드로 사용방법만 소개해볼까합니다.

```kotlin
interface Train {
    fun go(): String
}
```

```kotlin
@Module
abstract class MultiBindingModule {

    @Binds
    @IntoMap
    @CarKey(CarType.SUPER)
    abstract fun bindSuperTrain(train: SuperTrain): Train

    @Binds
    @IntoMap
    @CarKey(CarType.FLYING)
    abstract fun bindFlyingTrain(train: FlyingTrain): Train

    @Binds
    @IntoMap
    @CarKey(CarType.TOY)
    abstract fun bindToyTrain(train: ToyTrain): Train

}
```
위의 모듈에서는 Train을 구현한 3개의 다른 Train 구현체를 Binds 시키고 있습니다. @IntoMap 어노테이션을 이용하며 @CarKey라는 커스텀 어노테이션으로 Map의 key를 정의하고 있습니다. Dagger 자체적으로 @IntKey, @StringKey등의 미리 정의된 어노테이션이 있지만 조금더 깔끔한 코드를 위해 자체적으로 정의한 어노테이션을 사용하였습니다.

```kotlin
//CarType.kt
enum class CarType {
    SUPER, TOY, FLYING
}

//CarKey.kt
@Target(
        AnnotationTarget.FUNCTION, AnnotationTarget.PROPERTY_GETTER,
        AnnotationTarget.PROPERTY_SETTER
)
@Retention(AnnotationRetention.RUNTIME)
@MapKey
annotation class CarKey(val value: CarType)
```

 CarKey라는 어노테이션을 정의한 모습입니다. 대거에서 제공하는 @MapKey 어노테이션을 이용해 CarKey가 Multi Binding에서 사용되는 Key라는 점을 알려줘야 합니다.
 이제 뷰클래스에서 주입해보도록 하겠습니다.

```kotlin
class MainActivity : BaseActivity() {

    //컬렉션 주입 
    @Inject
    lateinit var train: Map<CarType, @JvmSuppressWildcards Train>

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_main)
        ...
        //미리 정의한 EnumType을 Key로 객체를 가져올 수 있다.
        findViewById<TextView>(R.id.text4).text = train[CarType.FLYING]?.go()
    }
}
```

위의 코드처럼 일반적인 Map을 사용하는 방식으로 객체를 가져올 수 있게 됩니다. 몇몇 블로그 포스팅에서는 이 처럼 Multi Binding을 활용하는 좋은 예제를 소개하고 있기도 하니 한번 공부해보는 것도 좋을 것 같습니다.

위의 예제들에서 살펴봤듯이 단순히 Module - Provides 조합만 사용한다면 Dagger를 사용할때 만들어야하는 코드량이 늘어나게 되지만 Constructor injection과 Binds를 적절히 활용한다면 불필요한 코드를 어느정도 제거할 수 있고 조금더 효율적으로 개발할 수 있을 것 같습니다.

모든 Sample 프로젝트는 github에 [Dagger2Android](https://github.com/iam1492/dagger2Android)에 올려두었습니다.