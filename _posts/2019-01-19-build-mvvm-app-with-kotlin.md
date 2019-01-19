---
layout: post
title:  "使用Kotlin构建MVVM应用程序—总览篇"
date:   2019-01-19 10:18:00
categories: Android Kotlin MVVM
---

说到MVVM，大家都会想起前端的MVVM框架，相较于前端MVVM的火热，它在移动开发领域就不那么热门了。Google在2015年才推出DataBinding框架，起步较晚，而且2015年是MVP模式爆发的一年，2016年是各种热修复、插件化爆发的一年，它没赶上好时机。

> PS：DataBinding和MVVM二者并不相同。MVVM是一种架构模式，而DataBinding是Android中实现数据与UI绑定的框架，是构建MVVM架构的一个工具，作用类似于增强版的ButterKnife。

自16年接触DataBinding以来，苦于这方面的知识较少，但是Databinding在使用过程中又十分便捷，所以一直以来都在不停探索怎样才能构建出合适的MVVM架构程序，在经过几次的项目重构之后，终于在近期结合Kotlin语言探索出了更适合Android的MVVM架构。

小专栏 ：[使用Kotlin构建Android MVVM应用程序](https://xiaozhuanlan.com/ditclear?rel=2493325177)

Github示例：<https://github.com/ditclear/PaoNet>

我们先来看看什么是MVVM，然后再一步一步来阐述整个的MVVM框架

### MVC、MVP、MVVM

我们先大致了解下Android开发的创建模式

#### MVC

> **Model**：实体模型、数据的获取、存储等等
>
> **View**：Activity、fragment、view、adapter、xml等等
>
> **Controller**：为View层处理数据，业务等等

从这个结构来看，Android本身还是符合MVC架构的。不过由于作为纯View的xml功能太弱，以及controller能提供给开发者的作用较小，还不如在Activity页面直接进行处理，但这么做却造成了代码大爆炸。一个页面逻辑复杂的页面动辄上千行，注释没写好的话还十分不好维护，而且难以进行单元测试，所以这更像是一个Model-View的架构，不适用于打造稳定的Android项目。

#### MVP

> **Model**：实体模型、数据的获取、存储等等
>
> **View**：Activity、fragment、view、adapter、xml等等
>
> **Presenter**：负责完成View与Model间的交互和业务逻辑，以回调返回结果。

前面说，Activity充当了View和Controller的作用， 造成了代码爆炸。而MVP架构很好的处理了这个问题。其核心理念是通过一个抽象的View接口（不是真正的View层）将Presenter与真正的View层进行解耦。Persenter持有该View接口，对该接口进行操作，而不是直接操作View层。这样就可以把视图操作和业务逻辑解耦，从而让Activity成为真正的View层。

这也是现今比较流行的架构，可是弊端也是有的。如果业务复杂了，也可能导致P层太臃肿，而且V和P层有一定耦合度，如果UI有什么地方需要更改，那么P层不只改一个地方那么简单，还需要改View的接口及其实现，牵一发动全身，运用MVP的同行都对此怨声载道。

#### MVVM

> **Model**：实体模型、数据的获取、存储等等
>
> **View**：Activity、fragment、view、adapter、xml等等
>
> **ViewModel**：负责完成View与Model间的交互和业务逻辑，基于DataBinding改变UI

MVVM的目标和思想与MVP类似，但它没有MVP那令人厌烦的各种回调，利用DataBinding就可以更新UI和状态，达到理想的效果。

##### 数据驱动UI

在使用MVC或MVP开发时，我们如果要更新UI，首先需要找到这个view的引用，然后赋予值，才能进行更新。在MVVM中，这就不需要了。MVVM是通过数据驱动UI的，这些都是自动完成。数据的重要性在MVVM架构中得到提高，成为主导因素。在这种架构模式中，开发者重点关注的是怎样处理数据，保证数据的正确性。

##### 关注点分离

常见的错误就是把所有代码都写在Activity或者Fragment中。任何跟UI和系统交互无关的事情都不应该放在这些类当中。尽可能让它们保持简单轻量可以避免很多生命周期方面的问题。MVVM架构模式下，数据和业务逻辑都处于ViewModel中，ViewModel只关心数据和业务，不需要直接和UI打交道，而Model只需要提供ViewModel的数据源，View则关心如何显示数据和处理与用户的交互。

通过以上简述和与MVC、MVP的对比，我们可以发现MVVM还是很有优势的，而如果再搭配Kotlin语言的话，可以说是如虎添翼了。

### 如何开始？

其实结构已经很清晰了，我们只需要做M-V-VM层各层应该做的事情，做到关注点分离。

**M层** 的关注点是怎么提供数据给ViewModel

**ViewModel层** 关注点是怎么处理数据（包括使用DataBinding绑定数据，以及控制loading、empty状态）

**View层**的关注点是显示数据，接收用户的操作，调用ViewModel中的方法

为了打造更适合Android的MVVM架构，使用到的技术有AOP、Dagger2、RxJava、Retrofit、Room和Kotlin，并遵循统一的命名规范和调用准则，保证开发时的一致性。

以下是我们现今的架构：

![MVVM](http://upload-images.jianshu.io/upload_images/3722695-70230207c39b8601.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)



### 创建文章详情界面

接下来我将展示一下M-V-VM三层之间如何协作，以文章详情页面为例

![](https://upload-images.jianshu.io/upload_images/3722695-6c178d5e810b8fb7?imageMogr2/auto-orient/strip%7CimageView2/2/w/400)

#### V—VM

UI由ArtcileDetailActivity.kt及article_detail_activity.xml组成。

要驱动UI，我们的数据模型需要持有几个元素：

- Article Id：文章详情的id，用于加载文章详情
- title：文章的标题
- content：文章的内容
- state：加载状态，用一个State类来封装

我们将创建一个ArticleDetailViewModel.kt来保存。

一个ViewModel为特定的UI组件提供数据，比如fragment 或者 activity，并负责和数据处理的业务逻辑部分通信，比如调用其它组件加载数据或者转发用户的修改。ViewModel并不知道View的存在，也不会受configuration change影响。

现在我们有了三个文件。

article_detail_activity.xml: 定义页面的UI

ArticleDetailViewModel.kt: 为UI准备数据的类

ArtcileDetailActivity.kt: 显示ViewModel中的数据与响应用户交互的控制器 

下面开始实现(为了简单，只显示了主要部分)：

```xml
<?xml version="1.0" encoding="utf-8"?>
<layout >

    <data>
        <import type="android.view.View"/>
        <variable
                name="vm"
                type="io.ditclear.app.viewmodel.ArticleDetailViewModel"/>
    </data>

    <android.support.design.widget.CoordinatorLayout>

        <android.support.design.widget.AppBarLayout>

            <android.support.design.widget.CollapsingToolbarLayout>
                <android.support.v7.widget.Toolbar
                        app:title="@{vm.title}"/>

            </android.support.design.widget.CollapsingToolbarLayout>
        </android.support.design.widget.AppBarLayout>

        <android.support.v4.widget.NestedScrollView>

            <LinearLayout>
                <ProgressBar
                        android:visibility="@{vm.loading?View.VISIBLE:View.GONE}"/>

                <WebView
                        android:id="@+id/web_view"
                        app:markdown="@{vm.content}"
                        android:visibility="@{vm.loading?View.GONE:View.VISIBLE}"/>

            </LinearLayout>
        </android.support.v4.widget.NestedScrollView>


    </android.support.design.widget.CoordinatorLayout>
</layout>
```

```kotlin
/**
 * 页面描述：ArticleDetailViewModel
 * @param repo 数据源Model(MVVM 中的M),负责提供ViewModel中需要处理的数据
 * Created by ditclear on 2017/11/17.
 */
class ArticleDetailViewModel(val repo: ArticleRepository) {

    //////////////////data//////////////
    lateinit var articleId:Int
    val loading=ObservableBoolean(false)
    val content = ObservableField<String>()
    val title = ObservableField<String>()

    //////////////////binding//////////////
    fun loadArticle():Single<Article> =
        repo.getArticleDetail(articleId)
                .async()
                .doOnSuccess { t: Article? ->
                    t?.let {
                        title.set(it.title)
                        content.set(it.content)

                    }
                }
                .doOnSubscribe { startLoad()}
                .doAfterTerminate { stopLoad() }



    fun startLoad()=loading.set(true)
    fun stopLoad()=loading.set(false)
}
```

```kotlin
/**
 * 页面描述：ArticleDetailActivity,处理和用户的交互（点击事件），以及处理
 * viewModel层回调的数据,附加一些显示Loading,空状态和绑定生命周期等等的操作
 * Created by ditclear on 2017/11/17.
 */
class ArticleDetailActivity : BaseActivity<ArticleDetailActivityBinding>() {

    override fun getLayoutId(): Int = R.layout.article_detail_activity

    @Inject
    lateinit var viewModel: ArticleDetailViewModel
  	
  	//init
  	override fun initView() {
       //统一都是KEY_DATA，别自己瞎命名
        val articleID: Int? = intent?.extras?.getInt(Constants.KEY_DATA)
        if (articleID == null) {
            toast("文章不存在", ToastType.WARNING)
            finish()
        }

        getComponent().inject(this)
      
        mBinding.vm = viewModel.apply {
            this.articleID = articleID
        }
    }
  
  	//加载数据
    override fun loadData() {

        viewModel.loadData()
                .compose(bindToLifecycle())
//              .doOnSubcribe{ showLoadingDialog() }
//              .doAfterTerminate{ hideLoadingDialog() }
                .subscribe({},{ dispatchFailure(it) })

    }

}
```

他们是如何工作的呢？

在进入到`ArticleDetailActivity`页面之后

1. init()方法->先进行数据的初始化，将viewModel和xml文件进行绑定
2. loadData()方法->调用viewModel的方法

进入`ArticleDetailViewModel`

1. 调用Model层获取详情方法获取数据源
2. 根据需要使用RxJava操作符对数据进行转换，通过DataBinding更新UI
3. 返回可观测的Single对象给View

回到`ArticleDetailActivity`页面

1. 绑定生命周期，避免内存泄漏
2. 对返回的可观测对象进行订阅
3. 处理成功和失败的情况

至此，V-VM之间如何协作就清楚了。

#### M—VM

现在我们把View和ViewModel联系了起来，但是ViewModel该如何获取数据呢？

我们使用Retrofit来从后端获取网络数据。

```kotlin
interface ArticleService{
    //文章详情
    @GET("article_detail.php")
    fun getArticleDetail(@Query("id") id: Int): Single<Article>
}
```

使用Room数据库来进行持久化

```kotlin
@Dao
interface ArticleDao{

    @Query("SELECT * FROM Articles WHERE articleid= :id")
    fun getArticleById(id:Int):Single<Article>


    @Insert(onConflict = OnConflictStrategy.REPLACE)
    fun insertArticle(article :Article)

}
```

然后使用ArticleRepository.kt对网络和本地操作进行一层封装

```kotlin
/**
 * 页面描述：ArticleRepository
 * 提供数据给ViewModel层 , 处理网络数据和本地缓存之间的关系
 * Created by ditclear on 2017/11/17.
 */
class ArticleRepository @Inject constructor
	(private val remote: ArticleService, private val local: ArticleDao) {

    /* 文章详情
     * 先查看本地是否有缓存，如果没有那么再去请求网络，成功后更新本地缓存
     */
    fun getArticle(articleId: Int): Single<Article> =
           local.getArticleById(articleId).onErrorResumeNext {
        if (it is EmptyResultSetException) {
            remote.getArticleDetail(articleId).doOnSuccess { t -> t?.let { local.insertArticle(it) } }
        } else throw it
    }
  
    }
```

先查看本地是否有缓存，如果没有那么再去请求网络，成功后更新本地缓存。

封装成Repository的原因是ViewModel不需要知道它的数据具体是从哪来的，这不是ViewModel这一层需要关心的事情。

即使你的项目没有进行数据缓存，总是从网络拉取数据，也建议封装成Repository，这意味着你的网络层是可以替换的，意义有点类似于封装一个ImageLoadUtil。

总体的流程就这么多，其实弄懂就很简单了。关键点是各层之间职责明确，以及解耦（Dagger2）和使用DataBinding时需要一个统一的规范。

而再细分，优化，也就是进行模块化、组件化的工作，深入些的插件化、热修复等等。不过万丈高楼平地起，我们的地基打的严实，以后的工作才会相对容易。

本文的代码都可以在<https://github.com/ditclear/PaoNet>中找到

### 一些建议

#### 建议一：在Activity或Fragment里处理点击事件

使用Presenter来继承View.OnClickListener

```kotlin
interface Presenter:View.OnClickListener{
    override fun onClick(v: View?)
}
```

然后在BaseActivity/BaseFragment里实现它

```kotlin
abstract class BaseActivity<VB : ViewDataBinding> : RxAppCompatActivity(),Presenter{
    
}
```

这样当我们要设置点击事件时，只需要

```kotlin
class ArticleDetailActivity : BaseActivity<ArticleDetailActivityBinding>() {
  
  	//...
 	//init
  	override fun initView() {

        mBinding.let{
            it.vm=mViewModel
          	it.presenter=this
        }
    }
}
```

在xml中使用时，则统一使用`presenter.onClick(view)`方法

```xml
<layout>
	<data>
  		<variable
                name="presenter"
                type="com.ditclear.paonet.view.helper.presenter.Presenter"/>
    </data>
  
  	<android.support.design.widget.CoordinatorLayout>
  		
      	<android.support.design.widget.FloatingActionButton
                android:id="@+id/stow_fab"                                                             				  
                android:onClick="@{(v)->presenter.onClick(v)}"
/>
  	</android.support.design.widget.CoordinatorLayout>
</layout>
```

真正处理则放在相应的Activity/Fragment里

```kotlin
class ArticleDetailActivity : BaseActivity<ArticleDetailActivityBinding>() {
  
  	//...
  	@SingleClick
  	override fun onClick(v: View?) {
        when (v?.id) {
            R.id.stow_fab -> stow()
          	//more ..
          	R.id.other_action -> other()
        }
    }
  //其它
   private fun stow() {
   }
  
  	//收藏
    private fun stow() {
        viewModel.stow().compose(bindToLifecycle())
                .subscribe({ toastSuccess(it?.message?:"收藏成功") }
                        , {  toastFailure(it) } })
    }
}
```

@SingleClick是一个注解，作为AspectJ的切面，来防止多次点击，需要将view作为参数，详细可参考文章

[DataBinding结合AspectJ防止多次点击](http://www.jianshu.com/p/ea45670a364f)

这是这样处理点击事件的原因之一，另一个好处是方便绑定生命周期，和进行回调处理(比如一些需要用到activity context的dialog和toast的时候，都可以写在doOnSubscribe和doAfterTerminate操作符里)，避免了ViewModel层持有context。

#### 建议二：多写单元测试

单元测试能保证数据和逻辑的正确性，而且语法相对简单，很容易学习。而且运行一次单元测试的时间简直毫秒杀运行一次app的时间。

我认为程序员和普通码农直接的区别之一便是是否进行单元测试。

而且由于ViewModel层是纯Kotlin/Java代码，感觉就如以前使用Eclipse写简单的控制台程序。

当然单元测试的作用不仅限于写测试代码，我一般都会在里面玩玩RxJava的操作符，进行一些算法的练习，验证数据的输出是否正确等等。

如果你想学习或了解单元测试，可以查看以下文章：

[关于安卓单元测试，你需要知道的一切(by 小创)](http://www.jianshu.com/p/dc30338a3e84)

[使用Kotlin和RxJava测试MVP架构的完整示例](http://www.jianshu.com/p/6d88998316b1) 

### 关于DataBinding

很多开发者放弃DataBinding原因就在于出错了不容易排查错误。
只显示出很多XXBinding未找到。
如果有一定使用经验的就知道只看最后一条报错信息就够了。
这里介绍一种我经常使用来排查错误的方式:
在Android Studio 的terminal 里运行

> ./gradlew clean assembleDebug

或者

> ./gradlew compileDebugJavaWithJavac

因为DataBinding是编译生成代码的，很多错误都是xml中表达式写的有问题导致的，所以运行以上命令容易在terminal中打印出具体错误的信息。这些命令对于需要编译生成代码的框架排查错误十分有用，比如Dagger2。

![](https://upload-images.jianshu.io/upload_images/3722695-9c346982aae23293.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

更多信息请查阅 [DataBinding实用指南](https://www.jianshu.com/p/015ad08c2c75)

### 规范

想要在使用DataBinding的过程中不出错，遵守统一的规范是一定的

1. **ViewModel—View—XML—Model(尽量)  应该相互对应,以功能模块开头**

**普通页面**

| ViewModel                 | View                     | XML                         |
| ------------------------- | ------------------------ | --------------------------- |
| ArticleDetailViewModel.kt | ArticleDetailActivity.kt | article_detail_activity.xml |

**列表页面** :请参考文章 [告别反复、冗余的自定义Adapter](https://www.jianshu.com/p/7d9dd93e9dd6)

| ViewModel               | View                   | XML                       |
| ----------------------- | ---------------------- | ------------------------- |
| ArticleListViewModel.kt | ArticleListActivity.kt | article_list_activity.xml |
| **Item ViewModel**      | **Item XML**           |                           |
| ArticleItemViewModel.kt | article_list_item.xml  |                           |

**Model 层命名**

| Remote            | Local         | Repository           |
| ----------------- | ------------- | -------------------- |
| ArticleService.kt | ArticleDao.kt | ArticleRepository.kt |

结构如下图所示：
![](https://upload-images.jianshu.io/upload_images/3722695-57a2fb96523461f7.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/400)

1. **xml布局文件中的variable统一命名**

   | ViewModel | Presenter(点击事件) | Item(列表项) |
   | --------- | ------------------- | ------------ |
   | vm        | presenter           | item         |

### 参考资料

[如何构建Android MVVM 应用框架](https://tech.meituan.com/android_mvvm.html)

[App开发架构指南（谷歌官方文档译文）