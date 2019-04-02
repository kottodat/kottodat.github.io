---
layout: posts
title: "[dagger2]프로젝트 틀 만들기(1)"
comment: true
tags: [DI, dagger2]
---

Application에 대거 입히기
===

##### 1. Application을 대장으로! AppComponent생성하기

이제부터 차차 Application의 Context를 활용한 객체들이나 정적함수 등을 만드는 것을 시작으로

프로젝트 전반적으로 활용 하도록 하는 틀을 만들어 가도록 하죠

ShopActivity 아닌 모든 객체들에게 적용 하도록 범위를 넓혀가보도록 해봅시다.

**이번 포스팅에서 할일**
- MyApplication생성
- AppModule추가
- CUComponent삭제 -> AppComponent 추가
- ActivityModule을 만들고 Activity마다 Module을 붙여준다.
- Inject(주입) 요청한 객체가 생성되는 위치 변경

---

### 이번 포스팅에서 추가로 적용하는 내용중 핵심 1

Application클래스를 중심으로 돌아 가고 Activity나 module들은 서브모듈과 다른 클래스에서 주입 하도록 한다.

자세한 내용은 [이 링크](https://rimduhui.tistory.com/57)를 타고 가서 한번 보고오자

요약하자면 **액티비티마다 inject하는 코드를 공통으로 넣기 싫어서** 정도가 되겠다.

---

### 이번 포스팅에서 추가로 적용하는 내용중 핵심 2

그리고 요 세가지에 대해서도 이야기 안하고 넘어갈 수가 없다.

- DaggerApplication
- AndroidInjector
- AndroidSupportInjectionModule

Application클래스는 **[DaggerApplication]** 을 상속받도록 해야하고
{% highlight language linenos %}
open class MyApplication : DaggerApplication()
{% endhighlight %}

Component는 **[AndroidInjector< T >]** 를 상속받고
(* 여기서 **T는 내가만든 Application클래스**)
{% highlight language linenos %}
interface AppComponent : AndroidInjector< MyApplication >
{% endhighlight %}

Component클래스의 상단에 모듈 명시하는 곳에 **[AndroidSupportInjectionModule]** 를 추가해준다.
{% highlight language linenos %}
@Component(modules = [ AndroidSupportInjectionModule::class,
{% endhighlight %}


[이 링크](https://android.jlelse.eu/new-android-injector-with-dagger-2-part-3-fe3924df6a89)에 가보면 설명이 있는데

번역이나 이해가 귀찮은 사람들을 위해 간단하게 요약하자면

상단에 있는 소스코드 영역 세개를 보면 점점 소스코드가 줄어드는 것을 볼 수가 있다.

**요약하면 대거2 써서 프래그먼트 다루다보면 액티비티마다 해줘야 하는 작업이 있는데 그걸 자동으로 해준다.**

---

**아래 적용한 예제코드가 다 있으니 실습하면서 보도록 하자**

**잊어버렸을까 해서 다시 한번 적는데**

**Inject요청 할 객체 -> 모듈 -> 컴포넌트 순으로 만든다.**

##### 0) Inject요청 할 클래스 생성

Context를 주입할거라 Context클래스는 따로 만들지 않는다.

##### 1) AppModule 생성

- 주로 application의 context를 필요로 하는 객체들을 제공(@Provides)하게 될것이다.
- 우선 applicationContext를 제공하는 메소드를 만들고 @Provides를 붙여주자

{% highlight language linenos %}
@Module
internal object AppModule {
    @Singleton
    @Provides
    @JvmStatic
    fun provideContext(application: Application): Context = application
}
{% endhighlight %}

##### 2) AppComponent 생성
- AndroidSupportInjectionModule을 모듈에 추가
- AppModule을 만들었으니 추가해주고
- HaagendazsModule은 삭제한다.
- AndroidInjector라는 녀석을 상속받도록 하고
- Builder도 정의해 주자
- **몇 가지는 당장 쓰지는 않는다. 일단 존잘님들이 쓰는 틀을 복붙 하는거다**

{% highlight language linenos %}
@Singleton
@Component(modules = [
    AndroidSupportInjectionModule::class,
    AppModule::class //,
    // HaagendazsModule::class 삭제
])
interface AppComponent : AndroidInjector<MyApplication> {
    @Component.Builder
    interface Builder {
        @BindsInstance
        fun application(application: Application): Builder
        fun build(): AppComponent
    }

    override fun inject(app: MyApplication)
}
{% endhighlight %}

##### 3) 어플리케이션
- DaggerApplication을 상속받자
- applicationInjector()라는 메소드를 오버라이드 해야하는데 내용은 아래와 같이 채워넣자
- **이것도 나중에 수정 추가 할 부분인데 일단 틀을 만들어 놓는다.**

{% highlight language linenos %}
open class MyApplication : DaggerApplication() {
    override fun applicationInjector(): AndroidInjector<out DaggerApplication> {
        return DaggerAppComponent.builder()
            .application(this)
            .build()
    }
}
{% endhighlight %}

**manifest에 application클래스 등록하는거 잊지말자**


##### 4) 메인 액티비티 수정

- component 만들어서 주입하는 코드도 액티비티마다 들어있고 보일러플레이트 코드 취급한다. 삭제한다.

{% highlight language linenos %}
class ShopActivity : DaggerAppCompatActivity() {

    @Inject
    lateinit var iscream: Iscream

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)

        // 아래 두줄은 삭제
        // var sampleComponent = DaggerCUComponent.create()
        // sampleComponent.inject(this)

        Toast.makeText(this, iscream.getName(), Toast.LENGTH_SHORT).show()

    }

}
{% endhighlight %}


##### 5) ActivityModule를 만든다.

- Activity마다 모듈을 만들어줄건데 그 모듈들을 관리할 모듈을 만든다.
- ShopActivityModule을 만들어서 ShopActivity 참조 할 수 있도록 한다.
- Iscream을 주입시켜본다.

{% highlight language linenos %}
@Module
abstract class  ShopActivityModule {

    @Binds
    abstract fun providesAppCompatActivity(shopActivity: ShopActivity): AppCompatActivity

    @Module
    companion object {
        @Provides
        @JvmStatic
        fun provideIscream(): Iscream {
            return VanillaIscream()
        }
    }
}
{% endhighlight %}

{% highlight language linenos %}
@Module
interface ActivityBindingMoudle
{
    @ContributesAndroidInjector(modules = [ShopActivityModule::class])
    fun shopActivity(): ShopActivity
}
{% endhighlight %}

만들었으면 AppComponent에 ActivityModule를 추가해주자
{% highlight language linenos %}
@Singleton
@Component(modules = [
    AndroidSupportInjectionModule::class,
    AppModule::class,
    ActivityBindingMoudle::class
])
interface AppComponent : AndroidInjector<MyApplication> {
{% endhighlight %}


##### 6) 실행 그리고 다시 수정

바닐라 아이스크림이 출력 될 것이다.

이전 포스팅과 다른점은 HaagendazsModule에서 주입받는게 아니라

ShopActivityModule에서 주입받는다는건데

HaagendazsModule을 참조해서 가져 오도록 수정 해보자.

{% highlight language linenos %}
@Module(includes = [HaagendazsModule::class]) /// 이부분이 추가
abstract class  ShopActivityModule {

    @Binds
    abstract fun providesAppCompatActivity(shopActivity: ShopActivity): AppCompatActivity

    /// 아이스크림 만드는 코드는 삭제했다.

}
{% endhighlight %}

---

dev와 prod를 각각 실행해보자

각각 바닐라 아이스크림과 초코 아이스크림이 출력 될 것이다.

여기까지 성공 했다면 이번 포스팅 목표도 달성했다고 볼 수 있다.

---
