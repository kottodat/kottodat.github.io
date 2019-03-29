---
layout: posts
title: 덩치를 키워보자
comment: true
tags: [DI][dagget2][dependency injection]
---

# 프로젝트 기본요소 구성해보기

***

##### dagger2를 활용하여 프로젝트 기본 요소들을 구성해보자

Activity와 Fragment를 몇 가지 추가하고

네트워크와 DB도 추가 해보도록 하자


1. 네트워크 붙여보기

안드로이드 스튜디오를 설치하면 안내해 주는 sunshine프로젝트에 보면 OpenWeather Api를 사용해서 통신하는 예제가 있으니

나도 OpenWeather api를 사용해서 통신 해보도록 하겠다.

계정 만들고 키 받는건 알아서 하도록 하자


1) 먼저 그래들에 dependency를 추가하자

{% highlight language linenos %}
/// okhttp
implementation 'com.squareup.okhttp3:logging-interceptor:3.9.1'

// retrofit
implementation 'com.squareup.retrofit2:retrofit:2.3.0'
implementation 'com.squareup.retrofit2:converter-moshi:2.3.0'
implementation 'com.squareup.retrofit2:adapter-rxjava2:2.3.0'
{% endhighlight %}



2) response객체 만들기

[OpenWeather Api](https://openweathermap.org/current) 여기서 response샘플을 복사해서

response객체를 만들도록 하자



3) opweather api interface만들기




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


---

dev와 prod를 각각 실행해보자

각각 바닐라 아이스크림과 초코 아이스크림이 출력 될 것이다.

여기까지 성공 했다면 이번 포스팅 목표도 달성했다고 볼 수 있다.

---
