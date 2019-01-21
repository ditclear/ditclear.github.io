---
layout: post
title:  "当Dagger2撞上ViewModel"
date:   2019-01-20 10:18:00
categories: Android dagger2 viewmodel
---

过去一年多的时间里，我一直在致力于打造一个最简单，并能让普通Android开发者都能快速上手的框架，并陆续发表了多篇开发心得，最终汇总为了[《使用Kotlin构建MVVM应用程序》](https://www.jianshu.com/p/da77266970d8)系列文章。其中就涉及到Dagger2和ViewModel的使用，这两者之间的碰撞令我想到了另一种十分简单的去进行依赖注入的可能，并引发了一系列的*化学反应*，可以说是天作之合。

> 可以在Github上查看相关代码：<https://github.com/ditclear/PaoNet>

**本文的写法不区分MVP还是MVVM结构，只是提供了一种不那么按部就班的注入方式。**

开始之前，我们先来了解一下Dagger2和ViewModel。

[**Dagger2**](https://www.jianshu.com/p/da77266970d8)是由Google提供的一个适用于Android和Java的快速的依赖注入工具，是现今众多Android开发者进行依赖注入的首选。

但由于其曲折的学习路线和较高的使用门槛，于是出现了一批又一批从入门到放弃的开发者，当然也包括我。

而[**ViewModel**](https://xiaozhuanlan.com/topic/6705498213)是Google的Jetpack组件中的一个。它是用来存储和管理UI相关的数据，将一个Activity或Fragment组件相关的数据逻辑抽象出来，并能适配组件的生命周期，**如当屏幕旋转Activity重建后，ViewModel中的数据依然有效**。它还可以帮助开发者轻易实现 **Fragment** 与 **Fragment** 之间, **Activity** 与 **Fragment** 之间的**通讯以及共享数据**。

我们可以通过以下的代码来获取ViewModel实例

```kotlin
 mViewModel=ViewModelProviders.of(this,factory).get(PaoViewModel::class.java)
```

其中要提供一个ViewModelProvider.Factory的实例来帮助构建你的ViewModel

```java
public interface Factory {
    /**
     * Creates a new instance of the given {@code Class}.
     * <p>
     *
     * @param modelClass a {@code Class} whose instance is requested
     * @param <T>        The type parameter for the ViewModel.
     * @return a newly created ViewModel
     */
    @NonNull
    <T extends ViewModel> T create(@NonNull Class<T> modelClass);
}
```

> PS：如果你使用的是MVP结构，那么只需要让其继承自ViewModel，也应该能达到相同的效果

### Dagger2？麻烦？

首先，我们先来看看Dagger2通常的依赖注入的方式

```java
public class FrombulationActivity extends Activity {
  @Inject Frombulator frombulator;

  @Override
  public void onCreate(Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);
    // DO THIS FIRST. Otherwise frombulator might be null!
    ((SomeApplicationBaseType) getContext().getApplicationContext())
        .getApplicationComponent()
        .newActivityComponentBuilder()
        .activity(this)
        .build()
        .inject(this);
    // ... now you can write the exciting code
  }
}
```

这是[Dagger-Android](https://google.github.io/dagger/android)用来吐槽Dagger2不行的示例，并给出了原因，这里我们也拿来用一次。

[Dagger-Android](https://google.github.io/dagger/android)给出了两点理由：

1. 只是复制粘贴上面的代码会让以后的重构比较困难，还会让一些开发者不知道Dagger到底是如何进行注入的（ps：然后就更不易理解了）
2. 更重要的原因是：它要求注射类型（FrombulationActivity）知道其注射器。 即使这是通过接口而不是具体类型完成的，它打破了依赖注入的核心原则：一个类不应该知道如何实现依赖注入。

**也就是说你就算是在基类(BaseActivity/BaseFragment)中将其封装一下，也无可避免的需要写**`getComponent.inject(this)`**这样的代码，而且还必须在对应的Component中添加相应的inject方法**，于是便有了以下的代码：

```kotlin
@ActivityScope
@Subcomponent
interface ActivityComponent {

    fun inject(activity: ArticleDetailActivity)

    fun inject(activity: CodeDetailActivity)

    fun inject(activity: MainActivity)

    fun inject(activity: LoginActivity)

    fun supplyFragmentComponentBuilder():FragmentComponent.Builder

}

@FragmentScope
@Subcomponent
interface FragmentComponent {

    fun inject(fragment: ArticleListFragment)
    fun inject(fragment: CodeListFragment)

    fun inject(fragment: CollectionListFragment)

    fun inject(fragment: MyCollectFragment)

    fun inject(fragment: HomeFragment)

    fun inject(fragment: RecentFragment)

    fun inject(fragment: SearchResultFragment)

    fun inject(fragment: RecentSearchFragment)

    fun inject(fragment: MyArticleFragment)

    @Subcomponent.Builder
    interface Builder {

        fun build(): FragmentComponent
    }
}
```

而目的也许就只是为了自动注入你的ViewModel或者Presenter对象，然后你的目录结构可能就会下图一般

![](https://upload-images.jianshu.io/upload_images/3722695-0203f870fe064f6e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/540)

而build之后生成的文件将会是这样的

![](https://upload-images.jianshu.io/upload_images/3722695-7e6319024e01754c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/340)

**然后就要用Dagger-Android来解决这些问题**？

是也不是，可能Dagger-Android解决了这些问题，但是它本身就比Dagger2更复杂，解决了这些问题，却引入了其它的问题，Android开发者并非都是Google开发者，不可能都具备这样强的逻辑和素质，实践之后我觉得还不如转向其它依赖注入的框架。

**我只是想注入一下我的ViewModel或Presenter，简简单单的开发，有必要这么麻烦吗？**

当然不是，也许我们并不需要Dagger-Android，**Dagger2**本身就能做到。

### 当Dagger2遇上ViewModel

配合[ViewModel](https://www.jianshu.com/p/627f0a9d8417)组件，我们根本不需要这么麻烦，而且也根本不需要再考虑注入到哪里去，在Component/Activity/Fragment中添加乱七八糟的`inject()`方法和`@Inject`。

我们只需要几个文件就好

![](https://upload-images.jianshu.io/upload_images/3722695-7ae57c68f152e684.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/640)

#### 怎么做？

通过`@Binds` 和`@IntoMap`

@Binds 和 @Provider的作用相差不大，区别在于@Provider需要写明具体的实现，而@Binds只是告诉Dagger2谁是谁实现的，比如

```kotlin
	@Provides
	fun provideUserService(retrofit: Retrofit) :UserService 	=retrofit.create(UserService::class.java)


    @Binds
    abstract fun bindCodeDetailViewModel(viewModel: CodeDetailViewModel):ViewModel


```

而@IntoMap则可以让Dagger2将多个元素依赖注入到Map之中。

```kotlin
/**
 * 页面描述：ViewModelModule
 *
 * Created by ditclear on 2018/8/17.
 */
@Module
abstract class ViewModelModule{

	// ...
    
    @Binds
    @IntoMap
    @ViewModelKey(CodeDetailViewModel::class)
    abstract fun bindCodeDetailViewModel(viewModel: CodeDetailViewModel):ViewModel

    @Binds
    @IntoMap
    @ViewModelKey(MainViewModel::class) //key
    abstract fun bindMainViewModel(viewModel: MainViewModel):ViewModel
 
    //...
    //提供ViewModel的工厂类
     @Binds
    abstract fun bindViewModelFactory(factory:APPViewModelFactory): ViewModelProvider.Factory
}
```

通过这些，Dagger2会根据这些信息自动生成一个关键的Map。key为ViewModel的Class，value则为提供ViewModel实例的Provider对象，通过`provider.get()`方法就可以获取到相应的ViewModel对象。

```java
private Map<Class<? extends ViewModel>, Provider<ViewModel>>
    getMapOfClassOfAndProviderOfViewModel() {
  return MapBuilder.<Class<? extends ViewModel>, Provider<ViewModel>>newMapBuilder(7)
      .put(ArticleDetailViewModel.class, (Provider) articleDetailViewModelProvider)
      .put(CodeDetailViewModel.class, (Provider) codeDetailViewModelProvider)
      .put(MainViewModel.class, (Provider) mainViewModelProvider)
      .put(RecentViewModel.class, (Provider) recentViewModelProvider)
      .put(LoginViewModel.class, (Provider) loginViewModelProvider)
      .put(ArticleListViewModel.class, (Provider) articleListViewModelProvider)
      .put(CodeListViewModel.class, (Provider) codeListViewModelProvider)
      .build();
}
```

而这些对象也是由Dagger2帮我们自动组装的。

![DaggerAppComponent](https://upload-images.jianshu.io/upload_images/3722695-14f9422b9398130b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

有了这些，我们就可以很方便的去构造ViewModel的工厂类`APPViewModelFactory`，并构造到所需的ViewModel。

```kotlin
/**
 * 页面描述：APPViewModelFactory  提供ViewModel 缓存的实例
 * 通过Dagger2将Map直接注入，通过key直接获取到相应的ViewModel
 * Created by ditclear on 2018/8/17.
 */
class APPViewModelFactory @Inject constructor(private val creators:Map<Class<out ViewModel>, @JvmSuppressWildcards Provider<ViewModel>>): ViewModelProvider.Factory{

    override fun <T : ViewModel?> create(modelClass: Class<T>): T {
        //通过class找到相应ViewModel的Provider
        val creator = creators[modelClass]?:creators.entries.firstOrNull{
            modelClass.isAssignableFrom(it.key)
        }?.value?:throw IllegalArgumentException("unknown model class $modelClass")
        try {
            @Suppress("UNCHECKED_CAST")
            return creator.get() as T //通过get()方法获取到ViewModel
        } catch (e: Exception) {
            throw RuntimeException(e)
        }
    }
}
```

到这里，ViewModel与Dagger2已经紧密联系起来，那如何不去写那么多恼人的`inject()`呢？

答案就是让你的`Application`持有你的`ViewModelProvider.Factory`实例，Talk is Cheap~

#### 在Application中进行注入

```kotlin
class PaoApp : Application() {

    @Inject
    lateinit var factory: APPViewModelFactory

    val appModule by lazy { AppModule(this) }

    override fun onCreate() {
        super.onCreate()
        //...
        DaggerAppComponent.builder().appModule(appModule).build().inject(this)
    }
}
```

#### 在Activity/Fragment之中使用

```kotlin
//基类BaseActivity
abstract class BaseActivity : AppCompatActivity(), Presenter {
	//...
    val factory:ViewModelProvider.Factory by lazy {
        if (application is PaoApp) {
            val mainApplication = application as PaoApp
           return@lazy mainApplication.factory
        }else{
            throw IllegalStateException("application is not PaoApp")
        }
    }
    
    fun <T :ViewModel> getInjectViewModel (c:Class<T>)= ViewModelProviders.of(this,factory).get(c)


    
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        //....

        initView()
    }
        abstract fun initView()
    //....
}
```

需要进行注入的Activity，比如ArticleDetailActivity就不需要再写@inject之类的注解

```kotlin
class ArticleDetailActivity : BaseActivity() {
	//很方便就可获取到ViewModel
	private val mViewModel: ArticleDetailViewModel by lazy { 	getInjectViewModel(ArticleDetailViewModel::class.java) }

	override fun initView() {
        //调用方法
   		mViewModel.dosth()
    }
}
```

Fragment相同的道理，具体可以查看[【PaoNet : Master分支】](https://github.com/ditclear/PaoNet)相应的代码。

### 写在最后

我们可以和通常的Dagger2、Dagger-Android的原理比较一下

- 普通的赋值：手动构造，十分繁琐，浪费时间

```kotlin
viewmodel = ViewModel(Repo(remote,local,prefrence))
```

- 通常的Dagger2注入：需要在Activity中用@Inject标识哪些需要被注入，并在Component中添加`inject(activity)`方法，会生成很多java类，有些繁琐

```kotlin
 instance.viewmodel = component.viewmodel
```

- Dagger-Android的注入：需要编写很多module，component，门槛高，不方便使用，还不如不用

```kotlin
app.map = Map<Class<? extends Activity>, Provider<AndroidInjector.Factory<? extends Activity>>>

activity.viewmodel = app.map.get(activity.class).getComponent().viewmodel
```

- Dagger2-ViewModel的注入：不需要在Activity中标识和inject，不会生成各种`XX_MemberInjectors`的java类，修改时改动最少，纯粹的一个**依赖检索容器**。

```kotlin
app.factory = component.AppViewModelFactory(Map<Class<? extends ViewModel>, Provider<ViewModel>>)

viewmodel = ViewModelProviders.of(this,app.factory).get(viewmodel.class)
```

对比Dagger-Android和Dagger2-ViewModel，两者都是间接通过Map来进行注入，不过一个的key是`Class<Activity>`，一个是`Class<ViewModel>`，而且都是在Application中inject一下。而Dagger2-ViewModel不需要向Dagger-Android那样添加`AndroidInjection.inject(this)`代码，更像是一个用来构造ViewModel的依赖管理容器，但对于我或者我希望打造的MVVM结构来说，这便已经足够了。

### 其它

代码地址：<https://github.com/ditclear/PaoNet>

《使用Kotlin构建MVVM应用程序系列》 ：<https://www.jianshu.com/c/50336d57e9b0







