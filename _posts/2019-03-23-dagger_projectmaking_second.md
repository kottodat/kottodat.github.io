---
layout: posts
title: 프로젝트 기반 다지기 - 1
comment: true
tags: [DI][dagget2]
---

#Dagger2 적용하여 프로젝트 기반 다지기 - 1

***
##### 1. Application을 대장으로! AppComponent생성하기

Application의 Context를 활용한 객체들이나 정적함수 등을 만들어서

프로젝트 전반적으로 활용 하도록 하고

MainActivity가 아닌 모든 객체들에게 적용 하도록 범위를 넓혀가보도록 하죠

**이번 포스팅에서 할일**
- MyApplication생성
- AppModule추가
- CUComponent -> AppComponent로 변경
- ActivityModule을 만들고 Activity마다 Module을 붙여준다.
- Inject(주입) 요청한 객체가 생성되는 위치 변경

Application클래스를 중심으로 돌아 가고 Activity나 module들은 서브모듈과 다른 클래스에서 주입 하도록 한다.

[이번 포스팅 내용에 대한 자세한 내용은 이 링크를 타고 가서 한번 보고오자](https://rimduhui.tistory.com/57)

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
- AndroidSupportInjectionModule이라는건 내가 모르는거지만 맨 위에 한줄 추가
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


##### 4) 메인 액티비티 수정

- component 만들어서 주입하는 코드도 액티비티마다 들어있고 보일러플레이트 코드 취급한다. 삭제한다.

{% highlight language linenos %}
class MainActivity : DaggerAppCompatActivity() {

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
- MainActivityModule을 만들어서 MainActivity가 참조 할 수 있도록 한다.
- Iscream을 주입시켜본다.

{% highlight language linenos %}
@Module
interface ActivityMoudle
{
    @ContributesAndroidInjector(modules = [MainActivityModule::class])
    fun mainActivity(): MainActivity
}
{% endhighlight %}

{% highlight language linenos %}
@Module
abstract class  MainActivityModule {

    @Binds
    abstract fun providesAppCompatActivity(mainActivity: MainActivity): AppCompatActivity

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

만들었으면 AppComponent에 ActivityModule를 추가해주자
{% highlight language linenos %}
@Singleton
@Component(modules = [
    AndroidSupportInjectionModule::class,
    AppModule::class,
    ActivityMoudle::class
])
interface AppComponent : AndroidInjector<MyApplication> {
{% endhighlight %}


##### 6) 실행 그리고 다시 수정

바닐라 아이스크림이 출력 될 것이다.

이전 포스팅과 다른점은 HaagendazsModule에서 주입받는게 아니라

MainActivityModule에서 주입받는다는건데 추상클래스에 저렇게 박아놓는건 좋지 않으니

dev와 prod에서 각각 지기 입맛대로 주입시키기 위해 좀더 수정을 하자

{% highlight language linenos %}
@Module(includes = [HaagendazsModule::class]) /// 이부분이 추가
abstract class  MainActivityModule {

    @Binds
    abstract fun providesAppCompatActivity(mainActivity: MainActivity): AppCompatActivity

    /// 아이스크림 만드는 코드는 삭제했다.

}
{% endhighlight %}

---

dev와 prod를 각각 실행해보자

각각 바닐라 아이스크림과 초코 아이스크림이 출력 될 것이다.

여기까지 성공 했다면 이번 포스팅 목표도 달성했다고 볼 수 있다.

---