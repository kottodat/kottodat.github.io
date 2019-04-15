---
layout: posts
title: "[dagger2]dagger2 시작하기"
comment: true
tags: [DI]
---

#[dagger2]dagger2 시작하기




### 시작하기 전에.....

**dagger2를 배우는 지인들을 위해 쓰고있는 글입니다.**

**많이 간소화된 내용으로 실습 해볼 수 있는 내용으로 쓰는 포스팅 입니다.**

**dagger2 관련 다른 글 몇개는 보고 오시고 TDD나 flavors를 모르면 좀 곤란합니다.**

**추가로 아래 링크들은 출퇴근 하면서 대충 한번 보시는 것을 강추합니다.**


**Dagger2(대거2) 관련**

- [New Android Injector with Dagger 2](https://medium.com/@iammert/new-android-injector-with-dagger-2-part-1-8baa60152abe)
- [Dependency Injection with Dagger 2](https://github.com/codepath/android_guides/wiki/Dependency-Injection-with-Dagger-2)
- [What is dependency injection?](https://stackoverflow.com/questions/130794/what-is-dependency-injection)
- [di with dagger2](https://speakerdeck.com/jakewharton/dependency-injection-with-dagger-2-devoxx-2014)

**flavors(플래버) 관련**

- [flavors(플래버) 안드로이드 공식문서](https://developer.android.com/studio/build/build-variants?hl=ko)

- [플래버 박상권님 포스팅](https://developer.android.com/studio/build/build-variants?hl=ko)




작업하면서 만든 **[소스코드 링크(github)](https://github.com/kottodat/dagger_test)** 를 받아서 보면서 하는 것을 추천

---

#### DI 간단 정리

- 내가 클래스 인스턴스를 만들지 않음

- 남이 만들어서 넣어주고 그걸 사용함

- new 하지 않는다. di한다.(직역하면 간접 주입 받는다?)

- **보통 Inject = '주입' 이라고 번역하는 듯 하다.**

---

우리를 괴롭힐 4가지 어노테이션 +@

- @Module
- @Provides
- @Inject
- @Component
- @????
- @...
- @...

4가지만 알면 코드짜서 돌려볼 수 있으니 다른건 나중에 생각 하도록 하자

### 실습 시작

시작전에 먼저 gradle에 dependency추가 등의 작업이 필요하다.

먼저 상단에 아래와 같이 한줄 추가해주고
{% highlight language linenos %}
apply plugin: 'kotlin-kapt'
{% endhighlight %}

dependencies에는

아래와 같이 추가해 준다.
{% highlight language linenos %}
implementation 'com.google.dagger:dagger-android:2.21'
implementation 'com.google.dagger:dagger-android-support:2.21'
kapt 'com.google.dagger:dagger-android-processor:2.21'

kapt 'com.google.dagger:dagger-compiler:2.21'
{% endhighlight %}


---

### 순서를 잘 기억해 두도록 하자

아래와 같이 세가지를 준비한다.

1) [Inject요청 할 객체]
2) [모듈]
3) [컴포넌트]

그 다음

1) [빌드] (빌드를 해야 Dagger{컴포넌트명} 클래스가 만들어진다.)
2) [Inject요청 할 곳에서 Dagger{컴포넌트명}.inject()호출]
3) @Inject를 붙여서 변수 선언

### 1. 주입당할 클래스 생성

이 클래스를 만드는데 new는 하지 않을것이다(물론 kotlin이라 new키워드는 원래 쓰지 않는다.)
{% highlight language linenos %}
class Iscream {
    fun getName(): String {
        return "아이스크림"
    }
}
{% endhighlight %}

### 2. 인스턴트 만들어줄 모듈

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

### 3. 주입 요청한 곳에 모듈을 연결시켜주는 컴포넌트

@Component는 클래스 위에 붙여주고 어떤 모듈들을 연결시켜 줄 것인지도 적어줍니다.

{% highlight language linenos %}
@Component(modules = [HaagendazsModule::class])
interface CUComponent {
    fun inject(activity: ShopActivity)

}
{% endhighlight %}

{: style="color:#FF0000; "}
\*주의\* inject의 인자는 꼭 ShopActivity 정확히 해당 클래스명을 넣어야함 AppCompatActivity같이 부모 클래스 넣으면 에러난다.
{: }

---

여기서 한번 빌드할것

빌드하면 DaggerCUComponent 클래스가 생성된다.

---

### 4. 주입을 요청

@Inject는 싶은 객체위에 선언하고 Component를 만들고 inject()를 호출하면 준비 완료

메소드를 호출해서 값이 제대로 얻어지는지 로그를 찍어봅니다.

{% highlight language linenos %}
class ShopActivity : AppCompatActivity() {

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

그냥 만들기만 하고 이해가 안되면 소용이 없으니 몇 장의 그림과 함께 간단한 설명을 해보겠다.

당신의 집앞에는 가게가 하나 있고 그 가게에서는 아이스크림을 구입 할 수 있다.

그 집에서는 아이스크림을 만들어서 판매하고 있다.

dagger2를 사용 할 경우와 그렇지 않을 경우를 각각 그림으로 비교하자면



![di-1-2](/assets/di-1-2.png)

{: style="text-align: center;"}
**< dagger2를 사용하지 않을 경우 >**
{: }


가게에서는 아이스크림을 직접 생산해서 판매 하는 것과 같고

아이스크림 틀의 크기나 틀에 뭘 넣느냐에 따라서 맛과 크기가 결정된다.

**new Iscream( "big", "choco", 2300 ) ;**

<br>




![di-1-3](/assets/di-1-3.png)

{: style="text-align: center;"}
**< dagger2를 사용 할 경우 >**
{: }

Dagger2를 도입 할 경우는 위 그림과 같이 아이스크림은 하겐다즈에서 생산하고

CU에서는 하겐다즈 코카콜라 돌레 등의 브렌드와 제휴하고 있으며

매장에서는 그냥 비치된 물건을 정해진 가격에 판매하면 된다.

**@Inject**
**Iscream iscream;**

<br/>


#### 다시한번 정리하자면...
@Module(HaagendazModule)은 아이스크림을 생산하는 브랜드고

@Component(CUComponent)는 브랜드들과 매장 사이의 중간다리 역활을 한다.

집 근처 Shop에서는 CU매장 오픈하고 아이스크림 판매대에 납품(@Inject)요청을 하면

가게를 열면 물건은 영업사원이 알아서 채워 넣는다.

**편의점에서는 아이스크림에 아무런 가공을 하지 않아도 냉장고에 하겐다즈 매대만 만들어 놓으면**

**영업 사원이 물건을 채워 넣을거고, 철이 지나면 포장지도 바뀌고 제품을 구성하는 원료의 종류나 비율도 개선 해 나갈것이다.**
