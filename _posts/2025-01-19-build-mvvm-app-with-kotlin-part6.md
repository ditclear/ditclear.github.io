---
layout: post
title:  "使用Kotlin构建MVVM应用程序—第六部分：LiveData"
date:   2019-01-19 10:18:00
categories: Android Kotlin MVVM
---

这里是使用 Kotlin 构建 MVVM 应用程序—第六部分：LiveData。

在前面的一系列文章中，我们了解了如何搭建一个 MVVM 架构的响应式 Android 应用程序。在写这一系列的过程中，有提议加入 LiveData 组件的，当时我的回复是：

![reason](https://upload-images.jianshu.io/upload_images/3722695-f05ec1c37a448fc7.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

可以看到我的解释是 LiveData 和要使用的 DataBinding 框架有冲突，而且当时使用 LiveData 还需要一个个的到 View 层去 Observe，太繁琐了，没有直接进行数据绑定那么方便，综合考虑之下，并没有将LiveData加入到本系列之中。

那为什么现在将 LiveData 搬上舞台了呢？当然是不冲突了呗。

至于原因可以看这篇文章：[Android开发利器之Data Binding Compiler V2 —— 搭建 Android MVVM 完全体的基础](http://tangpj.com/2018/05/12/data_binding_compiler_v2_1/)

简单来说，就是 google 升级了 DataBinding Compiler ，对 LiveData 进行了支持。

### 快速开始

要支持 LiveData，我们需要使用[DataBinding Compiler V2](https://developer.android.google.cn/topic/libraries/data-binding/start)，开启方式很简单

在你的`gradle.properties`文件中加入:

```groovy
android.databinding.enableV2=true
```

然后将以前的`ObserveField<*>`、ObserveBoolean 等等的代码都转变成`MutableLiveData<*>`的形式。

```kotlin
//////////////////data//////////////
val loading=MutableLiveData<Boolean>()
val content = MutableLiveData<String>()
val title = MutableLiveData<String>()
val error = MutableLiveData<Throwable>()

init {
    loading.postValue(false) // or loading.setValue(false)
    //getValue: title.value
}
```

LiveData赋值有两种方式`postValue`和`setValue`获取值则是直接使用`getValue()`，对于使用`ObserveField`比较熟练的同学，建议可以将其扩展为`ObserveField`的方式，也方便进行迁移改造。

```kotlin
fun  <T:Any> MutableLiveData<T>.set(value :T ?) = postValue(value)

fun  <T:Any> MutableLiveData<T>.get() = value

```

扩展之后，我们就不需要改动以前ViewModel层的代码

```kotlin
fun loadArticle():Single<Article> =
        repo.getArticleDetail(8773)
            .async(1000)
            .doOnSuccess { t: Article? ->
                t?.let {
                    title.set(it.title)
                    it.content?.let {
                        val articleContent=Utils.processImgSrc(it)
                        content.set(articleContent)
                    }

                }
            }
            .doOnSubscribe { startLoad()}
            .doAfterTerminate { stopLoad() }
```

在View层中，我们需要将当前`Activity/Fragment`的生命周期注入到生成的`Binding`对象中。

```kotlin
////binding
mBinding.vm=mViewModel
mBinding.setLifecycleOwner(this)//绑定生命周期，不写没效果
```

到此，我们的改造就已经完成了。

可以再看下`setLifecycleOwner(lifecycleOwner)`方法

```kotlin
@MainThread
public void setLifecycleOwner(@Nullable LifecycleOwner lifecycleOwner) {
    if (mLifecycleOwner == lifecycleOwner) {
        return;
    }
    if (mLifecycleOwner != null) {
        mLifecycleOwner.getLifecycle().removeObserver(mOnStartListener);
    }
    mLifecycleOwner = lifecycleOwner;
    if (lifecycleOwner != null) {
        if (mOnStartListener == null) {
            mOnStartListener = new OnStartListener(); //新建一个OnStartListener
        }
        lifecycleOwner.getLifecycle().addObserver(mOnStartListener); //注册一下
    }
    for (WeakListener<?> weakListener : mLocalFieldObservers) {
        if (weakListener != null) {
            weakListener.setLifecycleOwner(lifecycleOwner);//让LiveData感知生命周期
        }
    }
}
```

LiveDataListener.setLifecycleOwner(lifecycleOwner)

```
@Override
public void setLifecycleOwner(LifecycleOwner lifecycleOwner) {
    LifecycleOwner owner = (LifecycleOwner) lifecycleOwner;
    LiveData<?> liveData = mListener.getTarget();
    if (liveData != null) {
        if (mLifecycleOwner != null) {
            liveData.removeObserver(this);
        }
        if (lifecycleOwner != null) {
            liveData.observe(owner, this);
        }
    }
    mLifecycleOwner = owner;
}
```

### 兼容问题

> **Note:** The new data binding compiler in the Android Plugin version 3.1 is not backwards compatible. You need to generate all your binding classes with this feature enabled to take advantage of incremental compilation. However, the new compiler in the Android Plugin version 3.2 is compatible with binding classes generated with previous versions. The new compiler in version 3.2 is enabled by default.

DataBinding Compiler V2 在Android 插件 3.1版本是不向后兼容的，到了3.2版本才会完全兼容，并且会默认开启v2版本。

> 建议：到AS3.2版本再进行升级，避免无谓的修改

### 写在最后

这里只是简单介绍了从ObserveField转变到MutableLiveData的过程，但已经足够日常的开发。

LiveData解决了Data Binding不能感知View层生命周期的问题，更多的使用方法可以[查看官方文档](https://developer.android.google.cn/topic/libraries/data-binding/architecture)。

github:[https://github.com/ditclear/MVVM-Android/tree/livedata](https://github.com/ditclear/MVVM-Android/tree/livedata)

#### 参考资料：

[Android开发利器之Data Binding Compiler V2 —— 搭建Android MVVM完全体的基础](http://tangpj.com/2018/05/12/data_binding_compiler_v2_1/)

https://developer.android.google.cn/topic/libraries/data-binding/start