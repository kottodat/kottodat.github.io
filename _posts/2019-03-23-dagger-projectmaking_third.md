---
layout: posts
title: "[dagger2]프로젝트 기반 다지기 - 3"
comment: true
tags: [DI, dagger2]
---

Dagger2 적용하여 프로젝트 기반 다지기 - 3
===

##### DB도 추가해보자


##### 1. room 추가

room을 이용하여 로컬 데이터를 저장해보자

**1) 당연히 처음 할일은 그래들에 dependency 추가**

[공식 페이지](https://developer.android.com/jetpack/androidx/releases/room)에 있는 내용을 복붙했다.
{% highlight language linenos %}
/// room
def room_version = "2.1.0-alpha06"

implementation 'androidx.room:room-runtime:$room_version'
annotationProcessor "androidx.room:room-compiler:$room_version" // For Kotlin use kapt instead of annotationProcessor

// optional - Kotlin Extensions and Coroutines support for Room
implementation "androidx.room:room-ktx:$room_version"

// optional - RxJava support for Room
implementation "androidx.room:room-rxjava2:$room_version"

// optional - Guava support for Room, including Optional and ListenableFuture
implementation "androidx.room:room-guava:$room_version"

// Test helpers
testImplementation "androidx.room:room-testing:$room_version"

{% endhighlight %}

#####2) Entity 생성

[OpenWeather Api](https://openweathermap.org/current) 여기서 response샘플을 복사해서

[json to kotlin class](https://stackoverflow.com/questions/44180346/create-pojo-class-for-kotlin)을 참고해서

플러그인 설치하고 json데이터를 담을 response클래스를 만들자.

도중에 rain클래스 안에 변수명이 3h로 되서 에러가 나는데 rain항목은 일단 귀찮으니 주석처리 하자.

{% highlight language linenos %}
package com.kottodat.kottosample.data.api.response

data class ResByCoordinates(
    val clouds: Clouds,
    val cod: Int,
    val coord: Coord,
    val dt: Int,
    val id: Int,
    val main: Main,
    val name: String,
//    val rain: Rain,
    val sys: Sys,
    val weather: List<Weather>,
    val wind: Wind
)

data class Coord(
    val lat: Int,
    val lon: Int
)

data class Main(
    val humidity: Int,
    val pressure: Int,
    val temp: Double,
    val temp_max: Double,
    val temp_min: Double
)

data class Weather(
    val description: String,
    val icon: String,
    val id: Int,
    val main: String
)

//data class Rain(
//    val 3h: Int
//)

data class Wind(
    val deg: Double,
    val speed: Double
)

data class Clouds(
    val all: Int
)

data class Sys(
    val country: String,
    val sunrise: Int,
    val sunset: Int
)
{% endhighlight %}

#####3) retrofit interface를 만들자.

{% highlight language linenos %}
interface OpenWeatherMapApi {
    @GET("weather")
    @CheckResult
    fun byCoordinates(@Query("lat") lat: String, @Query("lon") lon: String):
            Single<ResByCoordinates>
}
{% endhighlight %}

#####4) retrofit객체를 다루는 모듈 생성
{% highlight language linenos %}
open class NetworkModule {

  companion object {
      val instance = NetworkModule()
  }

  @Singleton
  @Provides
  @IntoSet
  fun provideNetworkLogger(): Interceptor = HttpLoggingInterceptor().apply {
      level = HttpLoggingInterceptor.Level.BASIC
  }

  @Singleton
  @Provides
  fun provideOkHttpClient(
      loggingInterceptors: Set<@JvmSuppressWildcards
      Interceptor>
  ):
          OkHttpClient =
      OkHttpClient.Builder().apply {
          loggingInterceptors.forEach {
              addNetworkInterceptor(it)
          }
      }.build()

  @Singleton
  @Provides
  fun provideRetrofit(okHttpClient: OkHttpClient): Retrofit {
      return Retrofit.Builder()
          .client(okHttpClient)
          .baseUrl("https://samples.openweathermap.org/data/2.5/")
          .addConverterFactory(
              MoshiConverterFactory.create(
                  Moshi.Builder()
                      .build()
              )
          )
          .addCallAdapterFactory(RxJava2CallAdapterFactory.createAsync())
          .build()
  }

  @Singleton
  @Provides
  fun provideOpenWeatherMapApi(retrofit: Retrofit): OpenWeatherMapApi {
      return retrofit.create(OpenWeatherMapApi::class.java)
  }
}
{% endhighlight %}

모듈 만들었으면 컴포넌트에 등록하는거 잊지 말자

{% highlight language linenos %}
@Singleton
@Component(
    modules = [
        AndroidSupportInjectionModule::class,
        AppModule::class,
        NetworkModule::class,
        ActivityMoudle::class
    ]
)
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

##### 5) WeatherRepository 생성

아름다운 최종디자인은 아니지만 일단 아래와 같이 심플하게 만들어 놓고 가자.
다음 포스팅에 room적용하면 수정 할 예정이니 일단 가자
{% highlight language linenos %}
class WeatherRepository @Inject constructor(
    private val openWeatherMapApi: OpenWeatherMapApi
) {
    fun getSampleData(): Single<ResByCoordinates> =
        openWeatherMapApi.byCoordinates("37.277488", "127.0145353")
}
{% endhighlight %}


##### 6) MainActivity에서 테스트해보기
{% highlight language linenos %}
class MainActivity : DaggerAppCompatActivity() {

    @Inject
    lateinit var iscream: Iscream

    @Inject
    lateinit var weatherRepository: WeatherRepository


    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)

        Toast.makeText(this, iscream.getName(), Toast.LENGTH_SHORT).show()

        weatherRepository.getSampleData().subscribeOn(Schedulers.computation())
            .observeOn(AndroidSchedulers.mainThread())
            .subscribe({
                Toast.makeText(this, "성공", Toast.LENGTH_SHORT).show()
            }, {
                Toast.makeText(this, "실패", Toast.LENGTH_SHORT).show()
            })
    }

}
{% endhighlight %}

**manifest에 internet 권한 주는거 잊지 말자**


MainActivity의 weatherRepository위에 붙어있는 @Inject는 꼬리를 물고 물어서

Mainactivity.weatherRepository
-> com.kottodat.kottosample.di.AppModule.provideWeatherRepository(openWeatherMapApi)
-> com.kottodat.kottosample.di.NetworkModule.provideOpenWeatherMapApi(retrofit)
-> com.kottodat.kottosample.di.NetworkModule.provideRetrofit(okHttpClient)
-> com.kottodat.kottosample.di.NetworkModule.provideOkHttpClient(loggingInterceptors)

위와같은 순서로 타고타고타고 올라가게된다.

중간에 어디선가 끊기게 되면 아래와 같은 로그가 뜨게되니 어느 지점에서 주입 요청이 실패했는지 찾아 보도록 하자

com.kottodat.kottosample.scene.main.MainActivity.weatherRepository
com.kottodat.kottosample.scene.main.MainActivity is injected at
java.util.Set<okhttp3.Interceptor> is injected at
com.kottodat.kottosample.di.NetworkModule.provideOkHttpClient(loggingInterceptors)
okhttp3.OkHttpClient is injected at
com.kottodat.kottosample.di.NetworkModule.provideRetrofit(okHttpClient)
retrofit2.Retrofit is injected at
com.kottodat.kottosample.di.NetworkModule.provideOpenWeatherMapApi(retrofit)
com.kottodat.kottosample.data.api.OpenWeatherMapApi is injected at
com.kottodat.kottosample.di.AppModule.provideWeatherRepository(openWeatherMapApi)
com.kottodat.kottosample.data.WeatherRepository is injected at
com.kottodat.kottosample.scene.main.MainActivity.weatherRepository
com.kottodat.kottosample.scene.main.MainActivity is injected at
dagger.android.AndroidInjector.inject(T) [com.kottodat.kottosample.di.AppComponent → com.kottodat.kottosample.di.activitymodule.ActivityMoudle_MainActivity.MainActivitySubcomponent]


##### 7) NetworkModule의 싱글톤 처리
{% highlight language linenos %}
MyApplication클래스에 가서 DaggerAppComponent를 참조해서 소스를 따라가보면
아래와 같은 코드가 있다.

@Override
public AppComponent build() {
  if (networkModule == null) {
    this.networkModule = new NetworkModule();
  }
  Preconditions.checkBuilderRequirement(application, Application.class);
  return new DaggerAppComponent(networkModule, application);
}
{% endhighlight %}

bulid 할 때 new해서 넣어주고 있는데

다음과 같이 수정하자.

{% highlight language linenos %}
interface AppComponent : AndroidInjector<MyApplication> {
    @Component.Builder
    interface Builder {
        @BindsInstance
        fun application(application: Application): Builder
        fun networkModule(networkModule: NetworkModule): Builder // 라인 추가
        fun build(): AppComponent
    }
{% endhighlight %}

{% highlight language linenos %}
open class MyApplication : DaggerApplication() {
    override fun applicationInjector(): AndroidInjector<out DaggerApplication> {
        return DaggerAppComponent.builder()
            .application(this)
            .networkModule(NetworkModule.instance) /// 라인추가
            .build()
    }
}
{% endhighlight %}


다시 DaggerAppComponent를 까보면 다음과 같이 수정되어있는 것을 볼 수 있을것이다.
{% highlight language linenos %}
// 이런 메소드가 추가되어있음
@Override
public Builder networkModule(NetworkModule networkModule) {
  this.networkModule = Preconditions.checkNotNull(networkModule);
  return this;
}

// null이면 new하게 되어있으니 new하지 않는다.
@Override
public AppComponent build() {
  if (networkModule == null) {
    this.networkModule = new NetworkModule();
  }
  Preconditions.checkBuilderRequirement(application, Application.class);
  return new DaggerAppComponent(networkModule, application);
}
{% endhighlight %}

NetworkModule에 들어있는 instance를 사용하니 재생성 하지 않게 된다.

---

잘 되는지 테스트를 해보도록 하자

api key를 넣지 않아도 웹페이지 로드는 성공하고 내용이 실패라는 메세지가 뜨기 때문에

성공 토스트가 뜰 것이다.

[test](skrent://param?CUST_ID=201805160124,201812260343&CNTR_ID=201903120002)
[test2](skrent://camera?imgtype=01)



---
