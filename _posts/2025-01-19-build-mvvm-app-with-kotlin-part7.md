---
layout: post
title:  "使用Kotlin构建MVVM应用程序—第七部分：单元测试"
date:   2019-01-19 10:18:00
categories: Android Kotlin MVVM
---

这里是使用Kotlin构建MVVM应用程序—第七部分：单元测试。

 **单元测试 **这个词对于大多数android程序员来说应该是不陌生的，或者听说过，或者在某篇博客上见过，但是真正去实践过的可谓少之又少。

没实践的原因可能是：

- 业务繁重，没时间
- 没必要，测试的同事测过就可以了
- 需求变化快，写了也许又要改。。

总有理由安慰自己。那为什么我将其作为本系列的第六部分而非是提高篇里的内容呢？

> 在我看来，了解单元测试应该是每一名开发人员应该具备的素质，只有知道怎样的代码是适合进行单元测试的，才能写出高质量的代码。
>
> 可以简单的认为通过了单元测试的代码才是高质量的代码。

因此，我将其作为本系列的第六部分，希望学习本系列的android开发人员都能摆脱码农向工程师迈进，**不求掌握，但求了解**。

关于为什么要进行单元测试？还可以查看小创的文章[为什么要做单元测试](https://www.jianshu.com/p/68212278f592)

如果你想学习如何做单元测试，可以查看[关于安卓单元测试，你需要知道的一切](https://www.jianshu.com/p/dc30338a3e84)

### 在MVVM中如何进行单元测试？

首先，加入依赖

```groovy
//帮助进行mock
testImplementation 'org.mockito:mockito-core:2.15.0'
//单元测试
testImplementation 'junit:junit:4.12'
```

其次，知道要测试些什么？

[写点有价值的测试用例](https://www.jianshu.com/p/0429498d302b)这篇文章里对这个问题进行了解答

> 对于测试用例的设计，不能离开架构层面和业务层面

- **Presenter(ViewModel) 层**：这一层很清晰，我们为它的每个接口方法，以及每个方法里涉及的多个逻辑路径设计相应的测试用例，值得注意的是，这一层我们较少做输入输出的断言，而是验证是否正确覆盖V层和M层的逻辑。
- **Model层**: 同上，我们为它的每个方法设计测试用例，与P层不同，这一层要断言输入输出数据是否准确。
- **View层**：主要是进行ui测试是业务层面的测试。

那什么是**没价值的测试用例**，有以下几种：

1. 对成熟的工具类进行测试
2. 对简单的方法进行测试（比如get、set方法）
3. MVP(VM)各层重复测试，比如P(VM)层去断言输入输出的正确性

本文描述的单元测试主要是Model层和ViewModel层进行测试。

### Model层的单元测试

1. 快速创建测试文件

以`PaoRepo.kt`为例，在`PaoRepo`单词上按住`alt+enter`键即可快速创建对应的测试文件

![](https://upload-images.jianshu.io/upload_images/3722695-53b9ce04434c036a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/480)

![](https://upload-images.jianshu.io/upload_images/3722695-5837ffbef7a26aaa.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/480)

1. 写些什么

首先观察`PaoRepo.kt`

```kotlin
class PaoRepo constructor(private val remote: PaoService, private val local: PaoDao) {
	//获取文章详情
    fun getArticleDetail(id: Int) = local.getArticleById(id)
            .onErrorResumeNext {
                if (it is EmptyResultSetException) {
                    remote.getArticleById(id)
                            .doOnSuccess { local.insertArticle(it) }
                } else throw it
            }

}
```

构成一个`PaoRepo`对象需要通过构造方法传入一个`PaoService`和一个`PaoDao`对象。

由于我们只是测试逻辑，所以并不需要真实的去构造`PaoService`和`PaoDao`对象。这里我们就需要用到[Mockito](https://github.com/mockito/mockito)来进行mock。

```kotlin
class PaoRepoTest {

    private val local = Mockito.mock(PaoDao::class.java)
    private val remote = Mockito.mock(PaoService::class.java)
    private val repo = PaoRepo(remote, local)
    
}
```

当有了PaoRepo对象之后，我们开始对`getArticleDetail`方法的逻辑进行覆盖，而单元测试其实就是将这些测试用例翻译为计算机所知道的语句。

举几个例子：

- 当`local.getArticleById(id)`方法有数据返回的时候

  就不会抛出`EmptyResultSetException`异常，`remote.getArticleById(id)`和`local.insertArticle(it)` 都不会被调用

```kotlin
 	//mock返回数据
    private val article = mock(Article::class.java)
    //任意整数
    private val articleId = ArgumentMatchers.anyInt()

    @Test fun `local getArticleById`(){
        //当有数据返回的时候
        whenever(local.getArticleById(articleId)).thenReturn(Single.just(article))
        //进行方法模拟调用
        repo.getArticleDetail(articleId).test()
        //验证local.getArticleById(articleId)被调用
        verify(local).getArticleById(articleId)
        //验证remote.getArticleById(articleId)方法不被调用
        verify(remote, never()).getArticleById(articleId)
        //验证local.insertArticle()方法不被调用
        verify(local, never()).insertArticle(article)
    }
```

- 当本地数据库没找到数据，`local.getArticleById(1)`方法则会返回`EmptyResultSetException`异常，

  就会进入`onErrorResumeNext`代码块，由于是`EmptyResultSetException`异常，所以`remote.getArticleById(id)`和`local.insertArticle(it)` 都会被调用

```kotlin
@Test
fun `remote getArticleById`() {
    //当本地不能查到数据会抛出EmptyResultSetException
    whenever(local.getArticleById(articleId)).thenReturn(Single.error<Article>(EmptyResultSetException("本地没有数据")))
    //当调用remote.getArticleById(articleId)时返回数据
    whenever(remote.getArticleById(articleId)).thenReturn(Single.just(article))
    //进行方法模拟调用
    repo.getArticleDetail(articleId).test()
    //验证local.getArticleById(articleId)方法被调用
    verify(local).getArticleById(articleId)
    //验证remote.getArticleById(articleId)方法被调用
    verify(remote).getArticleById(articleId)
    //验证local.insertArticle(article)方法被调用
    verify(local).insertArticle(article)
}
```

运行以上单元测试

![](https://upload-images.jianshu.io/upload_images/3722695-4e2a86cd16748db4.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/720)

pass则代表逻辑已经成功覆盖，而且可以看到一共只需要315ms，如果要真机测试的话，光编译的时间就可能几分钟甚至十几分钟。

### ViewModel层的单元测试

首先看看`PaoViewModel.kt`

```kotlin
class PaoViewModel constructor(private val repo: PaoRepo) {

    //////////////////data//////////////
    val loading = ObservableBoolean(false)
    val content = ObservableField<String>()
    val title = ObservableField<String>()
    val error = ObservableField<Throwable>()

    //////////////////binding//////////////
    fun loadArticle(): Single<Article> =
            repo.getArticleDetail(8773)
                    .subscribeOn(Schedulers.io())
                    .delay(1000,TimeUnit.MILLISECONDS)
                    .observeOn(AndroidSchedulers.mainThread())
                    .doOnSuccess {
                        renderDetail(it)
                    }
                    .doOnSubscribe { startLoad() }
                    .doAfterTerminate { stopLoad() }


    fun renderDetail(detail: Article) {
            title.set(detail.title)
            detail.content?.let {
                val articleContent = Utils.processImgSrc(it)
                content.set(articleContent)
            }
    }


    private fun startLoad() = loading.set(true)
    private fun stopLoad() = loading.set(false)
}
```

通过上文的方法创建出对应的测试文件和数据mock之后，我们来覆盖`loadArticle()`方法的逻辑。

> ps：由于需要验证viewModel的方法是否有调用，我们需要使用Mockito.spy方法让viewModel对象可被侦察

```kotlin
class PaoViewModelTest {

    private val remote= mock(PaoService::class.java)

    private val local = mock(PaoDao::class.java)

    private val repo = PaoRepo(remote, local)

    private val viewModel = spy(PaoViewModel(repo))
}
```

- 当`repo.getArticleDetail()`方法请求成功之后，`renderDetail()`方法会被调用，当订阅开始时，loading的值为true，当订阅结束时，loading的值为false。

将上面👆的逻辑翻译为测试代码之后，如下所示：

```kotlin
 private val article = mock(Article::class.java)
@Before  //会在测试方法测试之前进行调用
fun setUp() {

    //让local.getArticleById()方法返回可观测的article
    whenever(local.getArticleById(anyInt())).thenReturn( Single.just(article))
}

@Test
fun `loadArticle success`() {
    
    //调用方法，进行验证
    viewModel.loadArticle().test()
    //验证加载中时loading为true
    Assert.assertThat(viewModel.loading.get(),`is`(true))
    //验证renderDetail()方法有调用
    verify(viewModel).renderDetail(article)
    //验证加载完成时loading为false
    Assert.assertThat(viewModel.loading.get(),`is`(false))

}
```

运行以上测试代码，会报`RuntimeException`.

![](https://upload-images.jianshu.io/upload_images/3722695-4fb4fe26f4723948.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/720)

看说明，应该是异步的时候会有问题。对于这样的情况，我们可以使用`RxJavaPlugins`和`RxAndroidPlugins`这些类来覆盖默认的`scheduler`。

为了便于复用到其它的测试类文件里，我们实现一个`TestRule`进行统一处理。

```kotlin
/**
 * 页面描述：ImmediateSchedulerRule
 * 使用RxJavaPlugins和RxAndroidPlugins这些类用TestScheduler覆盖默认的scheduler。
 * TestScheduler可以帮助我们控制时间来测试某些功能
 * Created by ditclear on 2018/11/19.
 */
class ImmediateSchedulerRule private constructor(): TestRule {

    private object Holder { val INSTANCE = ImmediateSchedulerRule () }

    companion object {
        val instance: ImmediateSchedulerRule by lazy { Holder.INSTANCE }
    }

    private val immediate = TestScheduler()

    override fun apply(base: Statement, d: Description): Statement {
        return object : Statement() {
            @Throws(Throwable::class)
            override fun evaluate() {
                RxJavaPlugins.setInitIoSchedulerHandler { immediate }
                RxJavaPlugins.setInitComputationSchedulerHandler { immediate }
                RxJavaPlugins.setInitNewThreadSchedulerHandler { immediate }
                RxJavaPlugins.setInitSingleSchedulerHandler { immediate }
                RxAndroidPlugins.setInitMainThreadSchedulerHandler { immediate }

                try {
                    base.evaluate()
                } finally {
                    RxJavaPlugins.reset()
                    RxAndroidPlugins.reset()
                }
            }
        }
    }
    //将时间提前xx ms
    fun advanceTimeBy(milliseconds:Long){
        immediate.advanceTimeBy(milliseconds,TimeUnit.MILLISECONDS)

    }
    //将时间提前到xx ms
    fun advanceTimeTo(milliseconds:Long){
        immediate.advanceTimeTo(milliseconds,TimeUnit.MILLISECONDS)

    }
}
```

有一点需要注意的是 我们需要将其设置为单例模式，否则会出现只有第一次测试才能成功，其它测试都失败的情况。

否则要解决这个问题，可能需要曲线救国，绕下弯路，通过[注入TestScheduler的方法](https://www.jianshu.com/p/0a845ae2ca64)来解决。具体问题可以查看笔者以前的译文[使用Kotlin和RxJava测试MVP架构的完整示例 - 第2部分](https://www.jianshu.com/p/0a845ae2ca64)

再运行这一单元测试，结果如下：

![](https://upload-images.jianshu.io/upload_images/3722695-2adb6493041abb47.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/720)

意思是`renderDetail()`方法未被调用。

这是正常的。仔细看代码就会发现这里有一个1000ms的延迟，而测试代码会顺序执行，不会像实际情况那样等待1000ms的延迟再去验证。

遇到这样的情况，我们就需要使用`TestScheduler`的`advanceTimeBy()`和`advanceTimeTo()`方法来控制时间。

更改后的测试代码如下所示：

```kotlin
@get:Rule
val testScheduler = ImmediateSchedulerRule.instance
@Before
fun setUp() {
    //让local.getArticleById()方法正常返回数据
    whenever(local.getArticleById(anyInt())).thenReturn( Single.just(article))
}
@Test
fun `loadArticle success`() {

    //调用方法，进行验证
    viewModel.loadArticle().test()
    //将时间提前500ms
    testScheduler.advanceTimeBy(500)
    //验证加载中时loading为true
    Assert.assertThat(viewModel.loading.get(),`is`(true))
    //由于有async(1000).1000毫秒的延迟，这里需要加快时间
    testScheduler.advanceTimeBy(500)
    //验证renderDetail()方法有调用
    verify(viewModel).renderDetail(article)
    //验证加载完成时loading为false
    Assert.assertThat(viewModel.loading.get(),`is`(false))

}
```

再运行一次测试代码：

![](https://upload-images.jianshu.io/upload_images/3722695-5004ab8ea59eee50.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/720)

### 编写方便进行单元测试的代码

通过以上的例子，我们了解了基础的单元测试该这么去写。

那怎么去方便写出这样的测试代码呢?

说到方便单元测试，这是很多人在写MVP和MVVM代码和贬低MVC时，基本都会说到的事情。

因为MVC的代码逻辑基本都糅合在Activity中，Activty就是MVC的`Controller`，如果将Activity中逻辑控制的代码提出到一个`Controller`之中，那也会出现和MVP/MVVM一样的三层结构。

但为什么MVC就不方便进行单元测试呢？

**最大的原因**就是**Controller中最好都要是纯Java或者纯Kotlin代码，不要导入有任何包含android包下的类，比如Context，View等**

**这些都不方便进行mock**，所以MVP结构就通过各种接口将逻辑代码和View层代码进行隔离，而在MVP的基础上通过数据绑定便成了MVVM。

**第二个要点**就是尽量遵从[面向对象六大原则](http://www.uml.org.cn/sjms/201211023.asp)中的单一职责原则，通过依赖注入来构造对象。

相信许多android开发者在开始编写android程序的初期，或多或少都写出过以下的代码。

```kotlin
class PaoViewModel  {

    //////////////////data//////////////
    val loading = ObservableBoolean(false)
    val content = ObservableField<String>()
    val title = ObservableField<String>()
    val error = ObservableField<Throwable>()

    //////////////////binding//////////////
    fun loadArticle(): Single<Article> =
            Repo().getArticleDetail(8773)//不通过注入直接new
                    .subscribeOn(Schedulers.io())
                    .delay(1000,TimeUnit.MILLISECONDS)
                    .observeOn(AndroidSchedulers.mainThread())
                    .doOnSuccess {
                        renderDetail(it)
                    }
                    .doOnSubscribe { startLoad() }
                    .doAfterTerminate { stopLoad() }
    
    fun otherAction() = Repo().otherAction()//不通过注入直接,再new一个
    
}
```

如果代码写成这样，试问如何通过**Mockito**来mock相应的行为呢？

而且这样的代码假如需要向Repo的构造方法中添加参数，那么修改量将是巨大的。

因此，尽量通过注入的方式进行参数注入而且也更符合开闭原则。

### 单元测试的旁门左道

在日常开发android的过程中，我们要验证自己的逻辑对不对，总是需要改动代码，然后运行程序，中间要build几分钟，然后如果结果不对，则又要反复这个过程。反反复复，一天就浪费过去了。

也许你只是想验证一下一个方法对不对？加一个0或者移动一下小数点？但是都会无谓的浪费时间。

这时候如果你知道单元测试的话，只需要在测试方法中验证一下输出就好了。

比如：BigDecimal(0.00)和BigDecimal(0.000)比较，是大？小？还是等于？

就可以编写一个单元测试，看看输出结果

```kotlin
class ExampleUnitTest{

    // if {@code this > val}, {@code -1} if {@code this < val},
    //         {@code 0} if {@code this == val}.
    @Test fun `test which is bigger `(){
        print(BigDecimal(0.00).compareTo(BigDecimal(0.000)))
    }
}
```

运行`test which is bigger`：

![](https://upload-images.jianshu.io/upload_images/3722695-6054b60648165956.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/720)

再一个好处就是方便你进行练习，比如Rxjava的操作符

```kotlin
@Test fun `practice rxJava operator`(){
    Single.just(2)
            .doOnSuccess {
                println("----------doOnSuccess--------")
            }
            .map { 3 }
            .doOnSubscribe {
                println("----------doOnSubscribe--------")
            }
            .doAfterTerminate {
                println("----------doAfterTerminate--------")
            }
            .subscribe({
                print("----------onSuccess --- $it-----")
            },{
                println(it.message)
            })
    
}
```

结果：

![](https://upload-images.jianshu.io/upload_images/3722695-1c5bf323cfffca22.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/720)

是不是想起了刚开始学习Java的时光。。

### 结尾

到此，我们对Model层和ViewModel层的单元测试就已经结束了。

由于篇幅原因，只进行了部分逻辑的覆盖，Model层的验证数据的输入输出正确与否并没有进行测试，如果想了解如何进行这方面的单元测试可以查看[GoogleSamples/android-architecture-components](https://github.com/googlesamples/android-architecture-components)的[**GithubBrowserSample**](https://github.com/googlesamples/android-architecture-components/tree/master/GithubBrowserSample)里的单元测试代码。

本文的重点不在于怎么进行单元测试，关于这一点，完全可以查看[关于安卓单元测试，你需要知道的一切](https://www.jianshu.com/p/dc30338a3e84)这篇文章。只希望能让跟随本系列学习MVVM结构的开发者了解单元测试，并且能编写出利于进行单元测试的代码。

所有的代码都可以在<https://github.com/ditclear/MVVM-Android> 中找到。

更多示例代码<https://github.com/ditclear/PaoNet>

#### 参考资料

[关于安卓单元测试，你需要知道的一切](https://www.jianshu.com/p/dc30338a3e84)

[【译】使用Kotlin和RxJava测试MVP架构的完整示例 - 第2部分](https://www.jianshu.com/p/0a845ae2ca64)

[android-architecture-components](https://github.com/googlesamples/android-architecture-components)





