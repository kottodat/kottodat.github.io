---
layout: posts
title: "[dagger2]flavors적용 프로젝트 나누기"
comment: true
tags: [DI, dagger2]
---

[dagger2]flavors를 써서 프로젝트 나누기
===


##### 1. 우선 Iscream을 interface로 만들자

<pre>
<code>
{% highlight language linenos %}
interface Iscream {
    fun getName(): String
}
{% endhighlight %}
</code>
</pre>



**첫 글에도 써놨지만 TDD라던가 flavors를 모르면 죰 곤란합니다.**
**일단 진행을 위해 아래 설정값을 복붙하고 첨부 스크린샷과 같이 패키지와 클래스를 구성합시다.**


##### 2. app build.gradle을 열고 flavors 셋팅을 해서 개발용과 배포용을 나눕니다.

개발용으로 dev
릴리즈용으로 prod

자세한 설정은 각자 알아서 하고 일단 최소한의 설정값은 아래 내용을 복붙하면 됩니다.

아주 간단한 복붙용 설정값
<pre>
<code>
{% highlight language linenos %}

flavorDimensions "flavors"
productFlavors
{
    dev
    {
        dimension "flavors"

    }
    prod
    {
        dimension "flavors"

    }
}

{% endhighlight %}
</code>
</pre>

당연히 dev와 prod폴더도 만들고 하위 패키지들도 생성해야하고

module과 주입 할 새로운 타입의 아이템인 VanillaIscream도 추가한다.

![di2-3](/assets/di-2-3.png)

#### 3. 배포용 prod도 설정하고 main에 있는 HaagendazsModule은 삭제

src->dev를 복사해서 src->prod도 만들어줍니다.

그리고 VanillaIscream은 ChocoIscream으로 바꿔줍니다.

그리고 src->main의 하위에 있는 di패키지에 HaagendazsModule은 삭제합니다.

![di-2-1](/assets/di-2-1.png)

위 이미지 처럼 모듈과 삽입 해줄 객체는 main은 가지고 있지 않고

dev와 prod패키지만 가지고 있습니다.

각각 모듈의 소스코드도 달라져야겠죠

바닐라 아이스크림 객체의 소스코드는 위 스크린샷 처럼 getName()메소드에서

"바닐라 아이스크림"을 리턴하고

초코아이스크림 객체의 소스는 "초코 아이스크림"을 리턴하도록 수정해 줍시다.

dev의 모듈의 소스코드
{% highlight language linenos %}
@Module
class HaagendazsModule {
    @Provides
    internal fun provideIscream(): Iscream {
        return VanillaIscream()
    }
}
{% endhighlight %}

prod의 모듈의 소스코드
{% highlight language linenos %}
@Module
class HaagendazsModule {
    @Provides
    internal fun provideIscream(): Iscream {
        return ChocoIscream()
    }
}

{% endhighlight %}

![di-2-2](/assets/di-2-2.png)

위 스크린샷 처럼 BuildVariants창을 열고

각각 두가지 버전으로 실행을 해봅시다.

prodDebug로 선택 후 실행

devDebug로 선택 후 실행

---

각각 초코 아이스크림과 바닐라 아이스크림 이라는 토스트가 뜨면

이번 포스팅의 목표도 달성!

---
