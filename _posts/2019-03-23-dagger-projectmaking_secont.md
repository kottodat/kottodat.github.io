---
layout: posts
title: "[dagger2]프로젝트 틀 만들기(2)"
comment: true
tags: [DI, dagget2]
---

##### Activity와 Fragment를 몇 가지 추가하고 네트워크와 DB도 추가 해보도록 하자

rxjava까지 사용해서 만들었다가 모른다고 해서 다시 고쳐서 쓴다 ㅜㅜ

##### 1. 네트워크 붙여보기

안드로이드 스튜디오를 설치하면 안내해 주는 sunshine프로젝트에 보면 OpenWeather Api를 사용해서 통신하는 예제가 있으니

나도 OpenWeather api를 사용해서 통신 해보도록 하겠다.

계정 만들고 키 받는건 알아서 하도록 하자

---

#### 작업 순서

retrofit사용하려면 보통 필요한게
- 호출 할 api들 정의해놓는 interface들
- 사용 할 request, response객체들
- 실제로 retrofit객체 생성하는 기능 담당하는 클래스
- 위 항목들을 모아서 만든 데이터 다루는 클래스

이정도 만든다.

우리가 dagger2를 곁들여서 할 작업 순서는

1) dependency추가
2) response받을 클래스 생성
3) api interface생성
4) 레트로핏 객체 생성하는 기능 담당하는 Module을 만듬
5) Module을 AppComopnent에 등록
6) Repository클래스 생성
7) AppModule에서 api interface를 주입 요청 (NetworkModule이 주입해줌)
8) ShopActivity에서 Repository를

거꾸로 이야기 하면
ShopActivity에서 Repository를 주입요청하면
AppModule에서 Provide해주고
provideWeatherRepository()는 인자가 있고 인자인 OpenWeatherMapApi도 주입받는데
이건 NetworkModule에 provideOpenWeatherMapApi()라는 메소드가 제공해준다.


---

#####1) 먼저 그래들에 dependency를 추가하자

okhttp, retrofit, ~~rxjava는 덤~~
{% highlight language linenos %}
/// okhttp
implementation 'com.squareup.okhttp3:logging-interceptor:3.9.1'

// retrofit
implementation 'com.squareup.retrofit2:retrofit:2.3.0'
implementation 'com.squareup.retrofit2:converter-moshi:2.3.0'
implementation 'com.squareup.retrofit2:adapter-rxjava2:2.3.0'

~~// rxjava
implementation 'io.reactivex.rxjava2:rxjava:2.1.9'
implementation 'io.reactivex.rxjava2:rxandroid:2.0.1'
implementation 'io.reactivex.rxjava2:rxkotlin:2.2.0'
implementation 'com.cantrowitz:rxbroadcast:2.0.0'~~
{% endhighlight %}

#####2) response객체를 만들도록 하자

[OpenWeather Api](https://openweathermap.org/current) 여기서 response샘플을 복사하고

[json to kotlin class](https://stackoverflow.com/questions/44180346/create-pojo-class-for-kotlin)을 참고해서

플러그인 설치하고 json데이터를 담을 response클래스를 만들자.

도중에 rain클래스 안에 변수명이 3h로 되서 에러가 나는데 rain항목은 일단 귀찮으니 주석처리 하자.

{% highlight language linenos %}
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
            Call<ResByCoordinates>
//    fun byCoordinates(@Query("lat") lat: String, @Query("lon") lon: String):
//            Single<ResByCoordinates>
}
{% endhighlight %}

#####4) retrofit객체를 다루는 모듈 생성
{% highlight language linenos %}
@Module
open class NetworkModule {

    companion object {
        val instance = NetworkModule()
    }

    @Singleton
    @Provides
    @IntoSet
    fun provideNetworkLogger(): Interceptor = HttpLoggingInterceptor().apply {
        level = HttpLoggingInterceptor.Level.BODY
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
                addNetworkInterceptor(interceptor)
            }
        }.build()

    @Singleton
    @Provides
    fun provideRetrofit(okHttpClient: OkHttpClient): Retrofit {
        return Retrofit.Builder()
            .client(okHttpClient)
            .baseUrl("https://samples.openweathermap.org/data/2.5/")
            .addConverterFactory(GsonConverterFactory.create())
            .build()

//            .addConverterFactory(
//                MoshiConverterFactory.create(
//                    Moshi.Builder()
//                        .build()
//                )
//            )
//            .addCallAdapterFactory(RxJava2CallAdapterFactory.createAsync())
//            .build()

    }

    val interceptor: Interceptor = Interceptor { chain: Interceptor.Chain ->
        val original = chain.request()
        var url: HttpUrl = original
            .url()
            .newBuilder()
            .addQueryParameter("APPID", "f05a608df7018dbb5d9c87b9b79ae42a")
            .build()
        val requestBuilder = original.newBuilder().url(url)
        chain.proceed(requestBuilder.build())

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
        ActivityBindingMoudle::class,
        DatabaseModule::class
    ]
)
interface AppComponent : AndroidInjector<MyApplication> {
    @Component.Builder
    interface Builder {
        @BindsInstance
        fun application(application: Application): Builder
        fun networkModule(networkModule: NetworkModule): Builder
        fun databaseModule(databaseModule: DatabaseModule): Builder
        fun build(): AppComponent
    }

    override fun inject(app: MyApplication)
}
{% endhighlight %}

##### 5) WeatherRepository 생성

아름다운 모양새는 아니지만 일단 아래와 같이 심플하게 만들어 놓고
다음 포스팅에 room적용하면 수정 할 예정이니 일단 가자
{% highlight language linenos %}
class WeatherRepository @Inject constructor(
    private val openWeatherMapApi: OpenWeatherMapApi
) {
    fun getSampleData(): Single<ResByCoordinates> =
        openWeatherMapApi.byCoordinates("37.277488", "127.0145353")
}
{% endhighlight %}

앱 모듈에서 Repository 주입 받을 수 있도록 @Provides붙여서 메소드 생성
{% highlight language linenos %}
@Module
internal object AppModule {
    @Singleton
    @Provides
    @JvmStatic
    fun provideContext(application: Application): Context = application

// 요 메소드를 추가한다.
    @Singleton
    @Provides
    @JvmStatic
    fun provideWeatherRepository(openWeatherMapApi: OpenWeatherMapApi, weatherRoomDatabase: WeatherRoomDatabase): WeatherRepository =
        WeatherRepository(openWeatherMapApi, weatherRoomDatabase)
}
{% endhighlight %}

##### 6) ShopActivity 테스트해보기
{% highlight language linenos %}
class ShopActivity : DaggerAppCompatActivity() {

    @Inject
    lateinit var iscream: Iscream

    @Inject
    lateinit var weatherRepository: WeatherRepository


    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)

        Toast.makeText(this, iscream.getName(), Toast.LENGTH_SHORT).show()

        weatherRepository.getSampleData(object : Callback<ResByCoordinates>
        {

            override fun onResponse(call: Call<ResByCoordinates>, response: Response<ResByCoordinates>) {
                Toast.makeText(this@ShopActivity, "성공", Toast.LENGTH_SHORT).show()
            }
            override fun onFailure(call: Call<ResByCoordinates>, t: Throwable) {
                Toast.makeText(this@ShopActivity, "실패", Toast.LENGTH_SHORT).show()
            }

        })
//            .andThen(performOtherOperation()) // a Single<OtherResult>
//            .subscribe({
//                Toast.makeText(this, "성공", Toast.LENGTH_SHORT).show()
//            }, {
//                Toast.makeText(this, "실패", Toast.LENGTH_SHORT).show()
//            })

    }

}
{% endhighlight %}

**manifest에 internet 권한 주는거 잊지 말자**


ShopActivity weatherRepository위에 붙어있는 @Inject는 꼬리를 물고 물어서

ShopActivity.weatherRepository
-> com.kottodat.kottosample.di.AppModule.provideWeatherRepository(openWeatherMapApi)
-> com.kottodat.kottosample.di.NetworkModule.provideOpenWeatherMapApi(retrofit)
-> com.kottodat.kottosample.di.NetworkModule.provideRetrofit(okHttpClient)
-> com.kottodat.kottosample.di.NetworkModule.provideOkHttpClient(loggingInterceptors)

위와같은 순서로 타고타고타고 올라가게된다.

---

**중간에 어디선가 끊기게 되면 아래와 같은 로그가 뜨게되니 어느 지점에서 주입 요청이 실패했는지 찾아 보도록 하자**

com.kottodat.kottosample.scene.main.ShopActivity.weatherRepository
com.kottodat.kottosample.scene.main.ShopActivity is injected at
java.util.Set<okhttp3.Interceptor> is injected at
com.kottodat.kottosample.di.NetworkModule.provideOkHttpClient(loggingInterceptors)
okhttp3.OkHttpClient is injected at
com.kottodat.kottosample.di.NetworkModule.provideRetrofit(okHttpClient)
retrofit2.Retrofit is injected at
com.kottodat.kottosample.di.NetworkModule.provideOpenWeatherMapApi(retrofit)
com.kottodat.kottosample.data.api.OpenWeatherMapApi is injected at
com.kottodat.kottosample.di.AppModule.provideWeatherRepository(openWeatherMapApi)
com.kottodat.kottosample.data.WeatherRepository is injected at
com.kottodat.kottosample.scene.main.ShopActivity.weatherRepository
com.kottodat.kottosample.scene.main.ShopActivity is injected at
dagger.android.AndroidInjector.inject(T) [com.kottodat.kottosample.di.AppComponent → com.kottodat.kottosample.di.activitymodule.ActivityMoudle_ShopActivity.ShopActivitySubcomponent]

---

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

AppComponent와 MyApplication의 일부를 다음과 같이 수정하자.

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

---
