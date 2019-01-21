---
layout: post
title:  "使用Kotlin构建MVVM应用程序—第一部分：入门篇"
date:   2019-01-19 10:18:00
categories: Android Kotlin MVVM
---

使用DataBinding已经有一年多的时间，Kotlin也写了好几个月了。在github上看了许多MVVM架构的项目（包括google的todo），但都没达到自己理想中的MVVM，可以说一千个人眼中就有一千个哈姆雷特，虽说都知道MVVM的概念，但是写法却是各不相同，有好有坏，结果越写越偏。到前段时间静下心来，开始使用kotlin，并遵循google的[App开发架构指南](http://www.jcodecraeer.com/a/anzhuokaifa/androidkaifa/2017/0523/7963.html)，才找到一种较好的构建MVVM应用程序的方式，使用kotlin来编写更是事半功倍，在这里分享给大家。
预计打算出4-5篇的样子，这是第一部分的入门篇，后续更新的话也会更新链接。
阅读正文之前，请先了解Databinding的基础语法，在这里可以进行学习[【Data Binding（数据绑定）用户指南】](http://www.jcodecraeer.com/a/anzhuokaifa/developer/2015/0606/3005.html)

开始正题。

### 首先：什么是MVVM？

MVVM是Model-View-ViewModel的简写，是有别于MVC和MVP的另一种架构模式。

相比于MVP，MVVM没有多余的回调，利用Databinding框架就可以将ViewModel中的数据绑定到UI上，从而让开发者只需要更新ViewModel中的数据，就可以改变UI。

![mvvm](http://upload-images.jianshu.io/upload_images/3722695-a112b0c94ddd1404?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

##### 再来讲一下分别的作用

- Model层：负责提供数据源给ViewModel，包含实体类，网络请求和本地存储等功能
- ViewModel：将Model层提供的数据根据View层的需要进行处理，通过DataBinding绑定到相应的UI上
- View：Activity、Fragment、layout.xml、Adapter、自定义View等等，负责将三者联系起来。


### 一个最基础的例子

这里我们定义一个实体`Animal.kt`

```kotlin
/**
 * 页面描述：Animal
 *
 * Created by ditclear on 2017/11/18.
 */
data class Animal(val name:String,var shoutCount:Int)
```

它有两个参数`name`和`shoutCount`。name代表动物的名称，shoutCount表示叫了几下。

看看我们需要做什么。。

![todo](http://upload-images.jianshu.io/upload_images/3722695-4e3383134ebfcc54?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

1. 展示需要的信息
2. 点击shout按钮时，shouCount+1并更新信息

很简单，几行代码就搞定了，先看看不用任何架构怎么做的

```kotlin
class AnimalActivityMVC : AppCompatActivity() {

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.animal_activity)
        val animal=Animal("dog",0)
        findViewById<TextView>(R.id.info_tv).text="${animal.name} 叫了 ${animal.shoutCount}声.."
        findViewById<View>(R.id.action_btn).setOnClickListener { 
            animal.shoutCount++
            findViewById<TextView>(R.id.info_tv).text="${animal.name} 叫了 ${animal.shoutCount}声.."
        }
    }
}
```

##### 再来看看MVVM怎么处理，以示区别

相比于前者，我们需要为`AnimalActivity`创建对应的`AnimalViewModel`

```kotlin
/**
 * 页面描述：AnimalViewModel
 * @param animal 数据源Model(MVVM 中的M),负责提供ViewModel中需要处理的数据
 * Created by ditclear on 2017/11/17.
 */
class AnimalViewModel(val animal: Animal){
    //////////////////data//////////////
    val info= ObservableField<String>("${animal.name} 叫了 ${animal.shoutCount}声..")
    //////////////////binding//////////////
    fun shout(){
        animal.shoutCount++
        info.set("${animal.name} 叫了 ${animal.shoutCount}声..")
    }
}
```

然后在View层的`AnimalActivity.kt`中将三者联系起来

```kotlin
class AnimalActivity : AppCompatActivity() {

    lateinit var mBinding : AnimalActivityBinding
    lateinit var mViewMode : AnimalViewModel

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        mBinding=DataBindingUtil.setContentView(this,R.layout.animal_activity)
        //////model
        val animal= Animal("dog", 0)
        /////ViewModel
        mViewMode= AnimalViewModel(animal)
        ////binding
        mBinding.vm=mViewMode
    }
}
```
目录架构现在是这个样子：
![](http://upload-images.jianshu.io/upload_images/3722695-affad38932643062.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/400)

Databinding强化了layout文件在Android中的地位，许多显示和点击事件都在xml中进行了处理。

```xml
<?xml version="1.0" encoding="utf-8"?>
<layout xmlns:android="http://schemas.android.com/apk/res/android"
        xmlns:tools="http://schemas.android.com/tools">

    <data>
        <!--需要的viewModel,通过mBinding.vm=mViewMode注入-->
        <variable
                name="vm"
                type="io.ditclear.app.viewmodel.AnimalViewModel"/>
    </data>

    <FrameLayout
            android:layout_width="match_parent"
            android:layout_height="match_parent"
            tools:context="io.ditclear.app.view.AnimalActivity">

        <TextView
                android:id="@+id/info_tv"
                android:layout_width="wrap_content"
                android:layout_height="wrap_content"
                android:text="@{vm.info}"
                tools:text="dog1叫了1声.."
                android:layout_marginBottom="24dp"
                android:layout_gravity="center"/>

        <Button
                android:id="@+id/action_btn"
                android:layout_width="wrap_content"
                android:layout_height="wrap_content"
                android:textAllCaps="false"
                android:text="shout"
                android:layout_marginTop="24dp"
                android:layout_gravity="center"
                android:onClick="@{()->vm.shout()}"/>

    </FrameLayout>
</layout>
```

可以看到`android:text="@{vm.info}"`和`android:onClick="@{()->vm.shout()}"`，分别调用了viewModel中的变量和方法。

所以大概的流程是

1. 用户点击按钮，调用`AnimalViewModel`的`shout`方法
2. ViewModel更新shoutCount和info数据,然后利用绑定自动更新了UI

流程很简单，但是反映了MVVM的思想，又有人会说，这样相比前者不是都多了那么多代码吗？

嗯，确实多了一个文件，但是却做到了关注点分离和数据驱动UI。

##### 借google的话来说

![架构准则](http://upload-images.jianshu.io/upload_images/3722695-8e0c7b3b2e231f13.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

另一个好处就是可以做单元测试，纯的kotlin代码写着再舒服不过，而且可以保证数据的正确性。相比于run app需要十几秒或者几分钟、十几分钟，run 一次单元测试是以毫秒记的，效率是很可观的。

### 结尾

github地址：[https://github.com/ditclear/MVVM-Android](https://github.com/ditclear/MVVM-Android)

这是使用Kotlin构建MVVM项目的第一部分，也是入门篇，所以很简单，介绍了一下MVVM的概念和基础写法，第二篇我将加入retrofit网络请求和Rxjava来谈谈如何较好的处理网络数据及绑定生命周期。