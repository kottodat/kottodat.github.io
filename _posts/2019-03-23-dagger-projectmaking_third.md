---
layout: posts
title: "[dagger2]프로젝트 틀 만들기(3)"
comment: true
tags: [DI, dagger2]
---

Dagger2 적용하여 프로젝트 기반 다지기 - 3


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

{% highlight language linenos %}
@Entity(tableName = "weather")
data class WeatherEntity(
    @PrimaryKey(autoGenerate = true)
    var id: Int = 0,
    @ColumnInfo(name = "humidity")
    var humidity: Int,
    @ColumnInfo(name = "pressure")
    var pressure: Int,
    @ColumnInfo(name = "temp")
    var temp: Double,
    @ColumnInfo(name = "itme")
    var itme: Long

)
{% endhighlight %}


#####3) Dao생성

{% highlight language linenos %}
@Dao
abstract class WeatherDao {

    @Insert(onConflict = OnConflictStrategy.REPLACE)
    abstract fun insertWeather(weather: WeatherEntity)

    @Query("DELETE FROM weather")
    abstract fun deleteAll()

    //    @Query("SELECT * FROM weather")
//    abstract fun getAllWeather(): Flowable<List<WeatherEntity>>
    @Query("SELECT * FROM weather")
    abstract fun getAllWeather(): List<WeatherEntity>
}
{% endhighlight %}

#####4) Database interface생성

{% highlight language linenos %}
interface WeatherDatabase {

    fun insertWeather(weather:WeatherEntity)
    fun deleteAll()
    fun getAllWeather(): List<WeatherEntity>
//    fun getAllWeather(): Flowable<List<WeatherEntity>>
}
{% endhighlight %}

#####5) Database interface를 상속받는 실 db객체 생성

{% highlight language linenos %}
class WeatherRoomDatabase @Inject constructor(
    private val database: RoomDatabase,
    private val weatherDao: WeatherDao
) : WeatherDatabase {
    override fun insertWeather(weather: WeatherEntity) {
        database.runInTransaction {
            weatherDao.insertWeather(weather)
        }
    }

    override fun deleteAll() {
        weatherDao.deleteAll()
    }

//    @CheckResult
//    override fun getAllWeather(): Flowable<List<WeatherEntity>> =
//        weatherDao.getAllWeather()
    @CheckResult
    override fun getAllWeather(): List<WeatherEntity> =
        weatherDao.getAllWeather()

}
{% endhighlight %}


#####6) Database를 생성하는 Module를 만들자.

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



#####7) AppComponent와 MyApplication도 수정하자

{% highlight language linenos %}
@Singleton
@Component(
    modules = [
        AndroidSupportInjectionModule::class,
        AppModule::class,
        NetworkModule::class,
        ActivityMoudle::class,
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

{% highlight language linenos %}
open class MyApplication : DaggerApplication() {
    override fun applicationInjector(): AndroidInjector<out DaggerApplication> {
        return DaggerAppComponent.builder()
            .application(this)
            .networkModule(NetworkModule.instance)
            .databaseModule(DatabaseModule.instance)
            .build()
    }
}
{% endhighlight %}

#####8) 마지막으로 Repository를 수정해보자.

서버에서 받아오면 그걸 room에 적재하고

성공이면 다시 room에서 적재된 리스트를 모두 가져오도록 하는 예제를 만들었다.

rxjava대신 AsyncTask와 Callable을 사용한 예제로 만들었다.

물론 나중에 고쳐서 써먹을 예정이다.

{% highlight language linenos %}
class WeatherRepository @Inject constructor(
    private val openWeatherMapApi: OpenWeatherMapApi,
    private val weatherRoomDatabase: WeatherRoomDatabase
) {

    fun getSampleData(onResult: (Boolean) -> Unit) {


        var call = openWeatherMapApi.byCoordinates("37.277488", "127.0145353")
        call.enqueue(object : Callback<ResByCoordinates> {
            override fun onResponse(call: Call<ResByCoordinates>, response: Response<ResByCoordinates>) {
                response.body()?.let {


                    AgentAsyncTask(it, weatherRoomDatabase, onResult).execute()

                }

            }

            override fun onFailure(call: Call<ResByCoordinates>, t: Throwable) {
                t.printStackTrace()
                onResult(false)
            }
        })
    }

    fun getLocalData(onResult: (List<WeatherEntity>) -> Unit) {
        onResult(getLocalData())

    }

    fun getLocalData(): List<WeatherEntity> {
        val callable: Callable<List<WeatherEntity>> = Callable<List<WeatherEntity>> {
            weatherRoomDatabase.getAllWeather()
        }
        val future = Executors.newSingleThreadExecutor().submit(callable)
        return future.get()
    }

    private class AgentAsyncTask(
        private val resByCoordinates: ResByCoordinates,
        private val weatherRoomDatabase: WeatherRoomDatabase,
        private val onResult: (Boolean) -> Unit
    ) : AsyncTask<Void, Void, Int>() {

        override fun doInBackground(vararg params: Void): Int? {
            weatherRoomDatabase.insertWeather(
                WeatherEntity(
                    humidity = resByCoordinates.main.humidity,
                    pressure = resByCoordinates.main.pressure,
                    temp = resByCoordinates.main.temp,
                    time = System.currentTimeMillis()
                )
            )
            return 0
        }

        override fun onPostExecute(agentsCount: Int?) {
            onResult(true)
        }
    }
}
{% endhighlight %}


#####9) 테스트를 해보자

{% highlight language linenos %}
class ShopActivity : DaggerAppCompatActivity() {

    @Inject
    lateinit var iscream: Iscream

    @Inject
    lateinit var weatherRepository: WeatherRepository


    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)

        Toast.makeText(this, iscream.getName(), Toast.LENGTH_SHORT).show()

        weatherRepository.getSampleData {
            if( it ) {
                weatherRepository.getLocalData {
                    Toast.makeText(this, "성공", Toast.LENGTH_SHORT).show()
                }
            } else {
                Toast.makeText(this, "실패", Toast.LENGTH_SHORT).show()
            }
        }


    }

}
{% endhighlight %}


---

리사이클러뷰에 데이터 출력하는 것 까지 해보려다가

답답해서 그냥 로그와 토스트로 되는지 여부 까지만 하기로 했다

다음 포스팅에서 mvvm패턴이랑 rxjava를 함께 버무려서 만들어 보도록 하자

---
