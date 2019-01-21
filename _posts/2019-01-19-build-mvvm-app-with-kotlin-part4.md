---
layout: post
title:  "使用Kotlin构建MVVM应用程序—第四部分：ViewModel"
date:   2019-01-19 10:18:00
categories: Android Kotlin MVVM
---

大家好，这里是使用Kotlin构建MVVM应用程序—第四部分：ViewModel。

本篇文章将介绍google推荐的架构组件ViewModel的使用方法及实现原理。

#### 为什么要有ViewModel?

为什么？看到ViewModel这个名字相信都会联系到MVVM架构中的VM。

但是在我看来，这两者并非是一个意思。如果你想实现MVVM架构的APP，按照

[使用Kotlin构建MVVM应用程序](12)基础篇的内容就已经足够了。

而我推测google把它称为ViewModel的原因可能有两点：

1. ViewModel架构组件是为VM层服务的。
2. 容易联想到MVVM架构，代表着google更推荐Android工程师们应用MVVM架构，而并非冗杂繁复的MVP。

当然这些是题外话。既然不使用ViewModel也能构建MVVM应用，那么ViewModel是来做什么的呢？

> The ViewModel class is designed to store and manage UI-related data in a lifecycle conscious way. The ViewModel class allows data to survive configuration changes such as screen rotations.

简单说来，ViewModel是用来存储和管理UI相关的数据，将一个Activity或Fragment组件相关的数据逻辑抽象出来，并能适配组件的生命周期，**如当屏幕旋转Activity重建后，ViewModel中的数据依然有效**。它还可以帮助开发者轻易实现 **Fragment** 与 **Fragment** 之间, **Activity** 与 **Fragment** 之间的**通讯以及共享数据**。

其中最具有吸引力的功能就是**屏幕旋转Activity重建后，ViewModel中的数据依然有效**。遥想当年，屏幕旋转当真是开发者不得不迈过的槛。而现在很多应用都只要求竖屏或者强制竖屏，不得横屏，所以对于这样的应用，我的建议是可以不用ViewModel组件，按照普通的VM开发就可以了。当然适当了解一下还是可以的。

#### ViewModel快速开始

首先我们需要在app/build.gradle加入相应的依赖

```groovy
//ViewModel
implementation "android.arch.lifecycle:extensions:1.1.1"
implementation "android.arch.lifecycle:viewmodel:1.1.1"
kapt "android.arch.lifecycle:compiler:1.1.1"
```

然后，让你的VM层继承ViewModel组件提供的`ViewModel`类，如果需要用到`Application`的`context`，那么就继承`AndroidViewModel`类。

```kotlin
class PaoViewModel constructor(private val repo: PaoRepo) :ViewModel(){
    //...
}
```

在View层的`PaoActivity`之中，以前的`mViewModel`是通过Dagger2注入的，而现在需要进行一下修改

```kotlin
class PaoActivity : AppCompatActivity() {

    lateinit var mBinding : PaoActivityBinding

    lateinit var mViewModel : PaoViewModel

    lateinit var factory: ViewModelProvider.Factory
	
     override fun onCreate(savedInstanceState: Bundle?) {
     	//....
        mViewModel=ViewModelProviders.of(this,factory).get(PaoViewModel::class.java)
        //...
     }
    
}
```

这里有两个概念`ViewModelProvider.Factory`和`ViewModelProviders`。再继续之前，我们需要先了解如何打造自己的`ViewModelProvider.Factory`。

#### ViewModelProvider.Factory

看到这个名字，自然就联想到工厂模式，这里提供我们需要的mViewModel实例。当然ViewModel没有那么神奇，不会帮我们自动生成，所以需要我们自己来实现需要的`ViewModelProvider.Factory`。

```kotlin
class PaoViewModelFactory : ViewModelProvider.Factory{
    //需要实现create方法，返回具体的viewmodel
    override fun <T : ViewModel?> create(modelClass: Class<T>): T {
        //return T
    }
}
```

那怎么创建具体的实例呢？可以在这里一个个new出相应的依赖

```kotlin

class PaoViewModelFactory  constructor(): ViewModelProvider.Factory{
    override fun <T : ViewModel?> create(modelClass: Class<T>): T {
        //////model
		val remote=Retrofit.Builder()
        	.baseUrl(Constants.HOST_API)
        	.addCallAdapterFactory(RxJava2CallAdapterFactory.create())
        	.addConverterFactory(GsonConverterFactory.create())
        	.build().create(PaoService::class.java)
		val local=AppDatabase.getInstance(applicationContext).paoDao()
		val repo = PaoRepo(remote, local)
		/////ViewModel
		mViewMode= PaoViewModel(repo)
        if (modelClass.isInstance(mViewMode)){
            return mViewMode as T
        }else{
            throw IllegalArgumentException("unknown model class $modelClass")
        }
    }

}
```

大功告成，现在我们的ViewModel已经具备了屏幕旋转Activity重建后，ViewModel中的数据依然有效的能力。

![rotate](https://user-gold-cdn.xitu.io/2018/7/4/164645fcd5d9eb36?w=320&h=523&f=gif&s=1513914)

#### 原理剖析

首先我们先来看看ViewModel

```java
public abstract class ViewModel {
    /**
     * This method will be called when this ViewModel is no longer used and will be destroyed.
     * <p>
     * It is useful when ViewModel observes some data and you need to clear this subscription to
     * prevent a leak of this ViewModel.
     */
    @SuppressWarnings("WeakerAccess")
    protected void onCleared() {
    }
}
```

非常简单，所以不可能是它变的魔术。

我们从入口处断点调试，看看到底是怎么发生的。

- ViewModelProviders.of(this,factory)

![ViewModelProviders](https://upload-images.jianshu.io/upload_images/3722695-cfad95ba6980ca39.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

 这里主要是构建了一个`ViewModelProvider`对象

- ViewModelStores.of(activity)

![ViewModelStores](https://upload-images.jianshu.io/upload_images/3722695-cc08f324fecb5586.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

这个ViewModelStore是什么呢？

```java
/**
 * Class to store {@code ViewModels}.
 * {@link android.arch.lifecycle.ViewModelStores} provides a {@code ViewModelStore} for
 * activities and fragments.
 */
public class ViewModelStore {

    private final HashMap<String, ViewModel> mMap = new HashMap<>();

    final void put(String key, ViewModel viewModel) {
        ViewModel oldViewModel = mMap.put(key, viewModel);
        if (oldViewModel != null) {
            oldViewModel.onCleared();
        }
    }

    final ViewModel get(String key) {
        return mMap.get(key);
    }

    /**
     *  Clears internal storage and notifies ViewModels that they are no longer used.
     */
    public final void clear() {
        for (ViewModel vm : mMap.values()) {
            vm.onCleared();
        }
        mMap.clear();
    }
}
```

很简单，就是一个用来贮存ViewModel实例的类，到这里大家大概也发觉了，其实也是一个map.get(key)的过程，key值是对应ViewModel的class。

```kotlin
mViewModel=ViewModelProviders.of(this,factory).get(PaoViewModel::class.java)
```

当然，仅此还不足以实现保留ViewModel数据的功能，注意`holderFragmentFor(activity)`

- holderFragmentFor(activity)

```java
/**
 * @hide
 */
@RestrictTo(RestrictTo.Scope.LIBRARY_GROUP)
public static HolderFragment holderFragmentFor(FragmentActivity activity) {
    return sHolderFragmentManager.holderFragmentFor(activity);
}
```

看来magic就在于这个`HolderFragment`了，不得不感叹Fragment的强大，虽然日常开发中不怎么好用，但是做一些小事情简直就是利器，再来看看HolderFragment做了什么。

- HolderFragment

```java
public class HolderFragment extends Fragment implements ViewModelStoreOwner {
    private static final String LOG_TAG = "ViewModelStores";
    //....
    private ViewModelStore mViewModelStore = new ViewModelStore();

    public HolderFragment() {
        setRetainInstance(true);
    }
    //...
      @Override
    public void onDestroy() {
        super.onDestroy();
        mViewModelStore.clear();
    }

    @NonNull
    @Override
    public ViewModelStore getViewModelStore() {
        return mViewModelStore;
    }
    //...
}

//A scope that owns {@link ViewModelStore}.
public interface ViewModelStoreOwner {
    /**
     * Returns owned {@link ViewModelStore}
     *
     * @return a {@code ViewModelStore}
     */
    @NonNull
    ViewModelStore getViewModelStore();
}


```

`HolderFragment`实现了`ViewModelStoreOwner`接口，并且提供了一个`ViewModelStore`对象，就如前文所说的那样，`ViewModelStore`贮存了我们所需的ViewModel。那它是怎么保证不重复创建，转屏时ViewModel中的数据依然可用呢？

熟悉Fragment的朋友相信已经明了了。

```java
 public HolderFragment() {
        setRetainInstance(true);
}
```

通过设置  `setRetainInstance(true)`。默认为false，设置为true后，屏幕旋转的时候Fragment不会重复创建，它里面的数据自然就还是以前的数据。

- .get(PaoViewModel::class.java)

![getByClass](https://upload-images.jianshu.io/upload_images/3722695-d0d36b4596855111.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

正如所说，不过这里的key加上了一个默认的名为DEFAULT_KEY的前缀。

```java
private static final String DEFAULT_KEY =
        "android.arch.lifecycle.ViewModelProvider.DefaultKey";
```

- get(key,modelClass)

```java
public <T extends ViewModel> T get(@NonNull String key, @NonNull Class<T> modelClass) {
    ViewModel viewModel = mViewModelStore.get(key);

    if (modelClass.isInstance(viewModel)) {
        //noinspection unchecked
        return (T) viewModel;
    } else {
        //noinspection StatementWithEmptyBody
        if (viewModel != null) {
            // TODO: log a warning.
        }
    }

    viewModel = mFactory.create(modelClass);
    mViewModelStore.put(key, viewModel);
    //noinspection unchecked
    return (T) viewModel;
}
```

通过map和key获取到相应的ViewModel，存在则返回，否则添加到ViewModelStore中。

![ViewModeStore](https://upload-images.jianshu.io/upload_images/3722695-d56aa2810b2d5a1a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

到此，我们的原理就已经剖析完成了。

至于为什么ViewModel组件可以帮助开发者轻易实现 **Fragment** 与 **Fragment** 之间, **Activity** 与 **Fragment** 之间的**通讯以及共享数据**？那是当然的啊，毕竟activity还有key值是一样的嘛。

#### ViewModel生命周期

![lifecycle](https://upload-images.jianshu.io/upload_images/3722695-3c847a780db2e5f3.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

通过上文的剖析之后，相信对这张图背后的东西有了更深的理解。

当`HolderFragment`的`onDestroy()`方法被调用的时候

```java
public void onDestroy() {
    super.onDestroy();
    mViewModelStore.clear();
}
```

再看看 `mViewModelStore.clear()`

```java
public final void clear() {
    for (ViewModel vm : mMap.values()) {
        vm.onCleared();
    }
    mMap.clear();
}
```

最后调用`ViewModel`的`onCleared()`方法。

#### 写在最后

本文基于ViewModel的1.1.1版本进行分析，以后的源码实现原理可能有变化。

另外ViewModel组件理解起来也不困难，主要是利用了Fragment的`setRetainInstance(true)`方法保证数据不被销毁和重复创建。

但唯一的遗憾便是为了提供一个ViewModel我们需要写太多模板化的代码了

```kotlin
//////model
val remote=Retrofit.Builder()
        .baseUrl(Constants.HOST_API)
        .addCallAdapterFactory(RxJava2CallAdapterFactory.create())
        .addConverterFactory(GsonConverterFactory.create())
        .build().create(PaoService::class.java)
val local=AppDatabase.getInstance(applicationContext).paoDao()
val repo = PaoRepo(remote, local)
/////ViewModel
mViewMode= PaoViewModel(repo)
```

如果能不写该多好。

> 上帝说：可以。

所以下一篇的内容便是依赖检索容器—Koin，实现ViewModel的快速注入

github:https://github.com/ditclear/MVVM-Android/tree/viewmodel

参考资料:

[google/viewmodel](https://developer.android.google.cn/topic/libraries/architecture/viewmodel)

[Android架构组件——ViewModel](https://blog.csdn.net/qq_24442769/article/details/79426609)（推荐）

