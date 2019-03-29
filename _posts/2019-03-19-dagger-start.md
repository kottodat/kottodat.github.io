---
layout: posts
title: "[dagger2]dagger2 시작하기"
comment: true
tags: [DI]
---
[dagger2]dagger2 시작하기
====


### 시작하기 전에.....

**dagger2를 배우는 지인들을 위해 쓰고있는 글입니다.**

이미 좋은 글들 아주아주 많은데 쉽게 실습 해볼 수 있는 포스팅이 찾기 힘들어서

대거2 적용이 너무 어렵다고 하는 분들을 위해 **실습 위주의 내용으로 포스팅** 합니다.

**dagger2 관련 다른 글 몇개는 보고 오시고 TDD나 flavors를 모르면 좀 곤란합니다.**

추가로 아래 링크들은 출퇴근 하면서 대충 한번 보시는 것을 강추합니다.


**Dagger2(대거2) 관련**

[Dependency Injection with Dagger 2](https://github.com/codepath/android_guides/wiki/Dependency-Injection-with-Dagger-2)
[What is dependency injection?](https://stackoverflow.com/questions/130794/what-is-dependency-injection)
[di with dagger2](https://speakerdeck.com/jakewharton/dependency-injection-with-dagger-2-devoxx-2014)

**flavors(플래버) 관련**

[flavors(플래버) 안드로이드 공식문서](https://developer.android.com/studio/build/build-variants?hl=ko)

[플래버 박상권님 포스팅](https://developer.android.com/studio/build/build-variants?hl=ko)


---

### DI 간단 정리

내가 클래스 인스턴스를 만들지 않음

남이 만들어서 넣어주고 그걸 사용함

new 하지 않는다. di한다.(직역하면 간접 주입 받는다?)

---

우리를 괴롭힐 4가지 어노테이션 +@
- @Module
- @Provides
- @Inject
- @Component
- @????
- @......


### 실습 시작
##### 1. 주입당할 클래스 생성

{% highlight language linenos %}
class Iscream {
    fun getName(): String {
        return "아이스크림"
    }
}
{% endhighlight %}

##### 2. 인스턴트 만들어줄 모듈

@Modlue은 클래스 위에

@Provides는 메소드 위에 붙여줍니다.

{% highlight language linenos %}
@Module
class HaagendazsModule {
    @Provides
    internal fun provideIscream(): Iscream {
        return Iscream()
    }
}
{% endhighlight %}

##### 3. 주입 요청한 곳에 모듈을 연결시켜주는 컴포넌트

##### <font color=FF0000>\*주의\* inject의 인자는 꼭 MainActivity처럼 정확히 명시해야함 AppCompatActivity같은거 넣으면 안된다.</font>

@Component는 클래스 위에 붙여주고 어떤 모듈들을 연결시켜 줄 것인지도 적어줍니다.

{% highlight language linenos %}
@Component(modules = [HaagendazsModule::class])
interface CUComponent {
    fun inject(activity: MainActivity)

}
{% endhighlight %}

##### 4. 주입을 요청

@Inject는 싶은 객체위에 선언하고 Component를 만들고 inject()를 호출하면 준비 완료

메소드를 호출해서 값이 제대로 얻어지는지 로그를 찍어봅니다.

{% highlight language linenos %}
class MainActivity : AppCompatActivity() {

    @Inject
    lateinit var iscream: Iscream

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)

        var sampleComponent = DaggerCUComponent.create()
        sampleComponent.inject(this)

        Toast.makeText(this, iscream.getName(), Toast.LENGTH_SHORT).show()
    }
}
{% endhighlight %}

***

여기까지 만들어서 토스트가 제대로 뜨면 Dagger2 첫 실습 성공

***
