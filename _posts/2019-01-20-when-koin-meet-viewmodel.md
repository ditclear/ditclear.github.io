---
layout: post
title:  "当Koin撞上ViewModel"
date:   2019-01-20 10:18:00
categories: Android koin viewmodel
---

在上一篇[《当Dagger2撞上ViewModel》](https://www.jianshu.com/p/d3c43b9dd6c6)的文章里，我简单阐述了Dagger-ViewModel这样的写法以简化Dagger2的使用，当时有评论推荐我使用Koin，当我尝试之后，发现Koin上手非常容易，实际上更加符合我的[《MVVM With Kotin》](https://www.jianshu.com/c/50336d57e9b0)框架，而且其也对ViewModel组件进行了支持，因此我在[PaoNet](https://github.com/ditclear/PaoNet)示例之中将Dagger2替换为了Koin，可以在以下链接中查看相关代码。

> 本文示例：<https://github.com/ditclear/MVVM-Android>
>
> 完整示例：<https://github.com/ditclear/PaoNet>

在这之间的迁移过程中，基本上没遇到什么大的问题，但也因为Koin上手比较容易，只有寥寥的几篇博客介绍了它的使用方法，对其原理介绍的还没看到。但作为开发者，如果只知道使用而不去了解它的内部实现，那么便会只知其形，而不解其意，当遇到问题会花费更多的时间去填坑，也浪费了一次提升自我能力的机会。

因此，在这里写下自己对Koin的一些心得体会，希望后来人能少走些弯路。

### What is Koin?

> A pragmatic lightweight dependency injection framework for Kotlin developers. Written in pure Kotlin using functional resolution only: no proxy, no code generation, no reflection!
>
> Koin is a DSL, a lightweight container and a pragmatic API.

[Koin](https://insert-koin.io/) 是为Kotlin开发者提供的一个实用型轻量级依赖注入框架，采用纯Kotlin 语言编写而成，仅使用功能解析，无代理、无代码生成、无反射。

Koin 是一个DSL，一个轻量级容器，也更加实用。

 开始之前，我们首先要知道几点知识。

- Inline Functions- [内联函数](http://www.kotlincn.net/docs/reference/inline-functions.html)

Koin使用了很多的内联函数，它的作用简单来说就是方便进行类型推导，能具体化类型参数

```kotlin
inline fun <reified T> membersOf() = T::class.members

fun main(s: Array<String>) {
    println(membersOf<StringBuilder>().joinToString("\n"))
}
```

- `module { }` - create a Koin Module or a submodule (inside a module)

类似于Dagger的@Module，里面提供所需的依赖

- `factory { }` - provide a *factory* bean definition 

类似于Dagger的@Provide，提供依赖，每次使用到的时候都会生成新的实例

- `single { }` - provide a bean definition

同factory，区别在于其提供的实例是单例的

- `get()` - resolve a component dependency

```kotlin
/**	 可通过name或者class检索到对应的实例
     * Retrieve an instance from its name/class
     * @param name
     * @param scope
     * @param parameters
     */
    inline fun <reified T : Any> get(
        name: String = "",
        scope: Scope? = null,
        noinline parameters: ParameterDefinition = emptyParameterDefinition()
    ): T = instanceRegistry.resolve(
        InstanceRequest(
            name = name,
            clazz = T::class,
            scope = scope,
            parameters = parameters
        )
    )
```

如果你想继续了解Koin，可以查看以下链接

> 官网：<https://insert-koin.io/>
>
> Koin-Dsl：<https://insert-koin.io/docs/1.0/quick-references/koin-dsl/>
>
> Koin文档：<https://insert-koin.io/docs/1.0/documentation/reference/index.html>

#### 快速上手

首先，添加Koin-ViewModel的依赖，注意需要对是否AndroidX版本进行区分

```gradle
// Add Jcenter to your repositories if needed
repositories {
    jcenter()
}
dependencies {
    // 非AndroidX 添加
    implementation 'org.koin:koin-android-viewmodel:1.0.1'
    // AndroidX 添加
    implementation 'org.koin:koin-androidx-viewmodel:1.0.1'
}
```

然后定义你所需的依赖定义的集合

```kotlin
val viewModelModule = module {
    viewModel { PaoViewModel(get<PaoRepo>()) }
    //or use reflection
//    viewModel<PaoViewModel>()

}

val repoModule = module {

    factory  <PaoRepo> { PaoRepo(get(), get()) }
    //其实就是
    //factory <PaoRepo> { PaoRepo(get<PaoService>(), get<PaoDao>())  }

	
}

val remoteModule = module {

    single<Retrofit> {
        Retrofit.Builder()
                .baseUrl(Constants.HOST_API)
                .addCallAdapterFactory(RxJava2CallAdapterFactory.create())
                .addConverterFactory(GsonConverterFactory.create())
                .build()
    }

    single<PaoService> { get<Retrofit>().create(PaoService::class.java) }
}


val localModule = module {

    single<AppDatabase> { AppDatabase.getInstance(androidApplication()) }

    single<PaoDao> { get<AppDatabase>().paoDao() }
}

//当需要构建你的ViewModel对象的时候，就会在这个容器里进行检索
val appModule = listOf(viewModelModule, repoModule, remoteModule, localModule)
```

在你的Application中进行初始化

```kotlin
class PaoApp : Application() {


    override fun onCreate() {
        super.onCreate()

        startKoin(this, appModule, logger = AndroidLogger(showDebug = BuildConfig.DEBUG))
    }


}
```

最后注入你的ViewModel

```kotlin
class PaoActivity : AppCompatActivity() {
    //di
    private val mViewModel: PaoViewModel by viewModel()
    
    //...
    
    fun doSth(){
        
        mViewModel.doSth()
     
    }
}
```

#### Koin是怎么进行注入的？

我们先撇开Koin的原理不谈，不用任何注入框架，这个时候，我们创建一个实例，就需要一步步的去创建其所需的依赖。

```kotlin
val retrofit = Retrofit.Builder()
        .baseUrl(Constants.HOST_API)
        .addCallAdapterFactory(RxJava2CallAdapterFactory.create())
        .addConverterFactory(GsonConverterFactory.create())
        .build()
val remote = retrofit.create(PaoService::class.java)
val database = AppDatabase.getInstance(applicationContext)
val local= database.paoDao()
val repo = PaoRepo(remote, local)
val mViewModel = PaoViewModel(repo)
```

当创建多个ViewModel的时候，这样子的模板化的代码无疑会拖慢开发效率。也正因为这些都是模板化的代码，创建方式都大体一致，因此便给了我们一种可能——**依赖检索**。

**假设我们有一个全局的容器，里面提供了应用所有所需实例的构造方式，那么当我们需要新建实例的时候，就可以直接从这个容器里面获取到它的构造方式然后拿到所需的依赖，从而构造出所需的实例。**

Koin要做的也就是这个。

当在Application中运行以下代码时

```kotlin
startKoin(this, appModule, logger = AndroidLogger(showDebug = BuildConfig.DEBUG))
```

如[Dagger-ViewModel](https://www.jianshu.com/p/d3c43b9dd6c6)所做的事情一样，Koin也会提供一个全局容器，将所有的依赖构造方式转换成`BeanDefinition`进行注册，这是一个HashSet，其名为`definitions`。

![definitions](https://upload-images.jianshu.io/upload_images/3722695-7b2ec18dd4cd33a6.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

而`BeanDefinition`得定义如下所示：

```kotlin
/**
 * Bean definition
 * @author - Arnaud GIULIANI
 *
 * Gather type of T
 * defined by lazy/function
 * has a type (clazz)
 * has a BeanType : default singleton
 * has a canonicalName, if specified
 *
 * @param name - bean canonicalName
 * @param primaryType - bean class
 * @param kind - bean definition Kind
 * @param types - list of assignable types
 * @param isEager - definition tagged to be created on start
 * @param allowOverride - definition tagged to allow definition override or not
 * @param definition - bean definition function
 */
data class BeanDefinition<out T>(
    val name: String = "",
    val primaryType: KClass<*>,
    var types: List<KClass<*>> = arrayListOf(),
    val path: Path = Path.root(),
    val kind: Kind = Kind.Single,
    val isEager: Boolean = false,
    val allowOverride: Boolean = false,
    val attributes: HashMap<String, Any> = HashMap(),
    val definition: Definition<T>
    )
```

我们主要看name以及primaryType，还记得`get()`关键字么？这两个便是依赖检索所需的key。

还有一个 `definition: Definition<T>`，它的值代表了其构造方式来源于那个module，对应前文的`viewModelModule`、`repoModule`、`remoteModule`、`localModule`，通过它可以反向推导该实例需要哪些依赖。

明白了这些，我们再来到获取ViewModel实例的地方，看看`viewModel()`方法是怎么做的。

```kotlin
class PaoActivity : AppCompatActivity() {
    //di
    private val mViewModel: PaoViewModel by viewModel()
   
}
```

`viwModel()`的具体实现

```kotlin
/**
 * Lazy getByClass a viewModel instance
 *
 * @param key - ViewModel Factory key (if have several instances from same ViewModel)
 * @param name - Koin BeanDefinition name (if have several ViewModel beanDefinition of the same type)
 * @param parameters - parameters to pass to the BeanDefinition
 */
inline fun <reified T : ViewModel> LifecycleOwner.viewModel(
    key: String? = null,
    name: String? = null,
    noinline parameters: ParameterDefinition = emptyParameterDefinition()
) = viewModelByClass(T::class, key, name, null, parameters)
```

默认通过Class进行懒加载，再来看看`viewModelByClass()`方法

```kotlin
/**
 * 获取viewModel实例
 *
 */
fun <T : ViewModel> LifecycleOwner.getViewModelByClass(
    clazz: KClass<T>,
    key: String? = null,
    name: String? = null,
    from: ViewModelStoreOwnerDefinition? = null,
    parameters: ParameterDefinition = emptyParameterDefinition()
): T {
    Koin.logger.debug("[ViewModel] ~ '$clazz'(name:'$name' key:'$key') - $this")

    val vmStore: ViewModelStore = getViewModelStore(from, clazz)
	//**关键在于这里**
    val viewModelProvider =
        makeViewModelProvider(vmStore, name, clazz, parameters)
	//ViewModel组件获取ViewModel实例
    return viewModelProvider.getInstance(key, clazz)
}

/**
 * 构建对应的ViewModelProvider
 *
 */
private fun <T : ViewModel> makeViewModelProvider(
    vmStore: ViewModelStore,
    name: String?,
    clazz: KClass<T>,
    parameters: ParameterDefinition
): ViewModelProvider {
    return ViewModelProvider(
        vmStore,
        object : ViewModelProvider.Factory, KoinComponent {
            override fun <T : ViewModel> create(modelClass: Class<T>): T {
                //在definitions中进行查找
                return get(name ?: "", clazz, parameters = parameters)
            }
        })
}
```

**文字描述一下其中的过程：**

比如现在需要一个PaoViewModel的实例，那么通过clazz为`Class<PaoViewModel>`的key在definitions中进行查找

![find in definitions](https://upload-images.jianshu.io/upload_images/3722695-842a3904b5c5ca74.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

最后查到有一个`PaoViewModel`的`BeanDefinition`，通过注册过的 `definition: Definition<T>`找到其构造方式的位置。

![发现ViewModel的构造方式](https://upload-images.jianshu.io/upload_images/3722695-2349d1143d7e79e5.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

当通过`PaoViewModel(get())`的构造方式去构造PaoViewModel实例的时候，发现又有一个`get<PaoRepo>()`，然后就是再重复前面的逻辑，一直到生成ViewModel实例为止。

这些通过Koin提供的Debug工具，可以在LogCat中很直观的看到构建过程。

![logcat](https://upload-images.jianshu.io/upload_images/3722695-0459ead5bf515ae1.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

而且报错更加友好，当你有什么依赖没有定义的时候，Koin也会比Dagger更好的提醒你。

### 写在最后

我们可以再跟Dagger-ViewModel比较一下。

两者构建实例的方法其实是一样的。

不同之处在于Koin需要我们定义好各个依赖它的构造方式，当我们需要具体实例的时候，它会去`definitions`容器里检索，逐步构造。

而Dagger-ViewModel则是通过注解，帮我们在编译期间就找到依赖，生成具体的构造方法，免去了运行时去检索的步骤。

**如果说把怎么样进行注入作为一道考题，那么这两者都可以算是正确答案。**

就实用性而言，我选择Koin，它是纯Kotlin代码，上手简单，而且不必在编译期间生成代码，减少了编译时间，报错也比Dagger2更加友好。再者，Koin还支持在构建过程中加入参数，是更适合我的依赖注入框架。

不过，Koin中有很多的内联函数和Dsl语法，源码中很多都没有明确的写明泛型，很容易把人看的云里雾里的，这也算是其缺点吧。

### 其它

Koin官网：<https://insert-koin.io/>

本文示例：<https://github.com/ditclear/MVVM-Android>

完整示例：<https://github.com/ditclear/PaoNet>

《使用Kotlin构建MVVM应用程序系列》 ：<https://www.jianshu.com/c/50336d57e9b0>



###  






