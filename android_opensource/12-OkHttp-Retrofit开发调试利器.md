> 前阵子定了个小目标，打算来深入了解下几个常用的开源库，看下其源码和实现原理，进行总结并输出成文章。初定的目标是 EventBus、ARouter、LeakCanary、Retrofit、Glide、OkHttp、Coil 等七个。**目前已经完成了十一篇关于 EventBus、ARouter、LeakCanary、Retrofit、Glide、OkHttp 的文章**，本篇是第十二篇，是关于 OkHttp 的实战内容，来实现一个 OkHttp 的可视化抓包工具，希望对你有所帮助😎😎

在使用 OkHttp 或者 Retrofit 的时候，我觉得大部分开发者会做得最多的自定义实现就是**拦截器**了。因为 OkHttp 的拦截器真的是太有用了，我们的很多需求：**添加 Header、计算并添加签名信息、网络请求记录**等都可以通过拦截器来自动完成，只要定义好规则，就可以覆盖到全局的 OkHttp 网络请求

按照我写 **[三方库源码笔记]** 这系列文章的习惯，我是每写一篇关于源码讲解的文章，就会接着写一篇关于该三方库的自定义实现或者是扩展阅读。所以，承接上一篇文章：[三方库源码笔记（11）-OkHttp 源码详解](https://juejin.im/post/6895369745445748749)  本篇文章就来写关于 OkHttp 的实战内容，来实现一个在移动端的可视化抓包工具：[Monitor](https://github.com/leavesC/Monitor)

### 一、Monitor 

Monitor 适用于使用了 **OkHttp/Retrofit** 作为网络请求框架的项目，只要添加了 **MonitorInterceptor** 拦截器，Monitor 就会自动记录并保存所有的**网络请求信息**且自动弹窗展示

最后实现的效果如下所示：

![](https://s1.ax1x.com/2020/10/21/BCJpz6.gif)

### 二、实现思路

这里来简单地介绍下 Monitor 的实现思路

其实 Monitor 是我蛮久前写的一个开源库了，刚好和我现在要写的文章主题相符，就趁着这机会做了一次整体重构，完全使用 Kotlin 语言来实现，请放心食用。其核心思路就是通过 Interceptor 拿到 Request 和 Response，记录各项请求信息，存到数据库中进行持久化保存，在实现思路上类似于 squareup 官方的`ogging-interceptor`，只是说 Monitor 会更加直接和方便😋😋

debug 版本的 MonitorInterceptor 的大体框架如下所示。HttpInformation 是对 request 和 response 的一个实体封装，也是最终会存到数据库中的实体类。通过 chain 拿到 request，先对本地数据库进行预先占位，在 proceed 后拿到 response，对本次请求结果进行解析，所有信息都存到 HttpInformation 中再来更新数据库，同时弹出 Notification

```kotlin
/**
 * 作者：leavesC
 * 时间：2020/10/20 18:26
 * 描述：
 * GitHub：https://github.com/leavesC
 */
class MonitorInterceptor(context: Context) : Interceptor {

    @Throws(IOException::class)
    override fun intercept(chain: Interceptor.Chain): Response {
        val request = chain.request()
        val httpInformation = HttpInformation()
        processRequest(request, httpInformation)
        httpInformation.id = insert(httpInformation)
        val response: Response
        try {
            response = chain.proceed(request)
        } catch (e: Throwable) {
            httpInformation.error = e.toString()
            throw e
        } finally {
            update(httpInformation)
        }
        processResponse(response, httpInformation)
        update(httpInformation)
        return response
    }

    private fun processRequest(request: Request, httpInformation: HttpInformation) {
        ···
    }

    private fun processResponse(response: Response, httpInformation: HttpInformation) {
        ···
    }

    private fun showNotification(httpInformation: HttpInformation) {
        ···
    }

}
```

release 版本的 MonitorInterceptor 则不会做任何操作，只是单纯将请求转发出去而已，不会造成多余的性能消耗和引用

```kotlin
class MonitorInterceptor(context: Context) : Interceptor {

    @Throws(IOException::class)
    override fun intercept(chain: Interceptor.Chain): Response {
        val request = chain.request()
        return chain.proceed(request)
    }

}
```

HttpInformation 包含了单次网络请求下所有关于 request 和 response 的请求参数和返回值信息，responseBody 只会保存文本类型的返回值（例如 Json 和 XML），图片这类二进制文件则不会进行保存

```kotlin
class HttpInformation {
    
    var url = ""
    var host = ""
    var path = ""
    var scheme = ""
    var protocol = ""
    var method = ""

    var requestHeaders = mutableListOf<HttpHeader>()
    var responseHeaders = mutableListOf<HttpHeader>()

    var requestBody = ""
    var requestContentType = ""
    var requestContentLength = 0L

    var responseBody = ""
    var responseContentType = ""
    var responseContentLength = 0L

    var requestDate = 0L
    var responseDate = 0L

    var responseTlsVersion = ""
    var responseCipherSuite = ""

    var responseCode = DEFAULT_RESPONSE_CODE
    var responseMessage = ""

    var error: String? = null
    
}
```

HttpInformation 则是用 Room 数据库来持久化保存，不得不说的是，Jetpack 中的 Room 和 LiveData 来搭配使用还是很爽的，将 LiveData 作为数据库的返回值，可以很方便地以观察者模式来实时监听数据库中的数据变化

```kotlin
/**
 * @Author: leavesC
 * @Date: 2020/11/14 16:14
 * @Desc:
 */
@Dao
interface MonitorHttpInformationDao {

    @Query("SELECT * FROM monitor_httpInformation WHERE id =:id")
    fun queryRecordObservable(id: Long): LiveData<HttpInformation>

    @Query("SELECT * FROM monitor_httpInformation order by id desc limit :limit")
    fun queryAllRecordObservable(limit: Int): LiveData<List<HttpInformation>>

    @Query("SELECT * FROM monitor_httpInformation order by id desc")
    fun queryAllRecordObservable(): LiveData<List<HttpInformation>>
    
}
```

UI 层则不用自己去考虑线程切换和内存泄露这类问题，直接进行 observe 即可

```kotlin
    private val monitorViewModel by lazy {
        ViewModelProvider(this).get(MonitorViewModel::class.java).apply {
            allRecordLiveData.observe(this@MonitorActivity, Observer {
                monitorAdapter.setData(it)
            })
        }
    }
```

### 三、远程引用

代码我已经发布到了 jitpack，方便大家直接远程依赖使用。同时引入 debug 和 release 版本的依赖，**release 版本的 MonitorInterceptor 不会做任何操作，避免了信息泄露，也不会增加 Apk 体积大小**

```groovy
        allprojects {
            repositories {
                maven { url 'https://jitpack.io' }
            }
        }

        dependencies {
           debugImplementation 'com.github.leavesC.Monitor:monitor:1.1.3'
           releaseImplementation 'com.github.leavesC.Monitor:monitor-no-op:1.1.3'
        }
```

只要向 OkHttpClient 添加了 MonitorInterceptor，之后的操作就都会自动完成

```kotlin
        val okHttpClient = OkHttpClient.Builder()
            .addInterceptor(MonitorInterceptor(Context))
            .build()
```

### 四、Github

GitHub 链接点击这里：[Monitor](https://github.com/leavesC/Monitor)