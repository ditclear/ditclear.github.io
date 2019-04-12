---
layout: post
title:  "聊聊MVX中的Model"
date:   2019-04-12 10:18:00
categories: Android 架构
---

随着Android架构的不断演进，从最初的MVC到MVP再到MVVM，变化的只有M和V层之间的部分，M和V层开发者似乎都已经统一了意见。

- Model 层 ： 实体模型、数据的获取、存储等等
- View层：向用户展示UI及处理交互

但据我在GitHub上看过的各种项目代码而言，许多人仅仅停留在字面上的理解，而没有真正的处理好三层间的边界。

今天，我们来聊一聊MVX中的Model。

### Model层

以`MVVM`架构为例，`Model`层的职责主要是**数据的获取和存储**然后将数据返回给`ViewModel`层。

![](https://upload-images.jianshu.io/upload_images/3722695-87bff6f4f9da6dd6.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/600)

在上面的图解中，我们分离出了一个仓库类`Repository`，这是Google开发架构指南中推荐的做法。

对此，在指南中这样解释：

> ViewModel的一个简单的实现方式是直接调用Webservice获取数据，然后把它赋值给User对象。虽然这样可行，但是随着app的增大会变得难以维护。ViewModel的职责过多也违背了前面提到的关注点分离（separation of concerns）原则。另外，ViewModel的有效时间是和[Activity](https://link.jianshu.com/?t=https://developer.android.com/reference/android/app/Activity.html)和[Fragment](https://link.jianshu.com/?t=https://developer.android.com/reference/android/app/Fragment.html)的生命周期绑定的，因此当它的生命周期结束便丢失所有数据是一种不好的用户体验。相反，我们的ViewModel将把这个工作代理给**Repository**模块。

因此，Repository 就成了一个很关键的模块，我们在这里处理所有关于数据的事情，包括

1. 从SharedPreference或者数据库或者服务器获取数据

1. 使用SharedPreference或者数据库缓存数据

1. 对请求参数的处理以及将返回数据处理为ViewModel层希望的类型

接下来，我们围绕着这三点来聊聊Model。

### 推荐的做法

- ##### 推荐 Model层通过SharedPreference存取数据

在我看过的大部分代码中，包括我自己以前也并没有意识到应该在Model层中通过SharedPreference存取数据，原因可能是没这个意识或者是因为写在其它地方也没影响到流程。

比如我们需要根据`SharedPreference`中缓存的用户Id来加载用户的详细信息，并且将返回结果也缓存在`SharedPreference`中。这种场景下经常会出现如下的代码：

```kotlin
/// View
final userId = spUtil.getString("userId")

viewmodel.getUserDetail(userId)
.subscribe({
 	//成功之后缓存详情
    spUtil.putString("userDetail")
},{})

/// ViewModel
fun getUserDetail(userId:String):Observable = repository.getUserDetail(userId)

/// Model
class UserRepository constructor(private val remote){
    /// 获取用户详情
	fun getUserDetail(userId:String):Observable = remote.getUserDetail(userId)
}
```

但是`SharedPreference`的作用其实类似于数据库，如果DB应该位于Model层，那么`SharedPreference`也同样，而且也不会出现参数传来传去的情况，改造之后的代码如下：

```kotlin
/// View
viewmodel.getUserDetail()
.subscribe({
 	//success
},{})

/// ViewModel
fun getUserDetail():Observable = repository.getUserDetail()

/// Model
class UserRepository constructor(private val remote:UserService,private val spUtil:SpUtil){
    /// 获取用户详情
    ///
    /// 通过 [spUtil] 拿到缓存的userId,获取到详情后再用[spUtil]进行缓存
	fun getUserDetail():Observable {
    	final userId = spUtil.getString("userId")
    	return remote.getUserDetail(userId).doOnSuccess{
        	spUtil.putString("userDetail")
    	}
	}
}
```

- ##### 推荐尽量在Repository中管理数据

我常常看到一些开发者只是把Repository当作是一个摆设，或者是一个象征性的东西，而没有实质上发挥它该有的作用。

比如有这样一个场景：展示文章详情，如果文章以前已被缓存过，那么直接获取缓存的数据，否则拉取服务端的数据并缓存到本地数据库。

就会出现如下的代码：

```kotlin
/// ViewModel
fun getArticleDetail(articleId:String):Observable {
    return repository.getLocalArticleById(articleId)
    				 .onErrorResumeNext {
						repository.getArticleById(id)
                        .doOnSuccess { repository.insertArticle(it) }
                     }.doOnSuccess{
                         // 数据转换成View层需要的数据
                         renderUI(it)
                     }
}

/// Model
class ArticleRepository constructor(private val remote:ArticleService,private val local:ArticleDao){
	/// 根据[articleId]从数据库中查找文章详情
    fun getLocalArticleById(articleId:String) = local.getLocalArticleById(articleId)
    
    /// 根据[articleId]从服务端获取文章详情
    fun getArticleById(articleId:String) = remote.getArticleById(articleId)
    
    /// 将文章详情[article]插入本地数据库
    fun insertArticle(Article article) = local.insertArticle(article)
}

```

这样的代码最大的问题是没做到**关注点分离**。

ViewModel层并不关心数据怎么来，也不关心数据应该怎么存储。它只关心拿到Model层的原始数据之后应该怎么将其转换为View层需要展示的数据。

基于这个原则，改造之后的代码如下：

```kotlin
/// ViewModel

fun getArticleDetail(articleId:String):Observable {
    return repository.getArticleDetail(articleId)
    				 .doOnSuccess{
                         // 数据转换成View层需要的数据
                         renderUI(it)
                     }
                                                  
 }

/// Model
class ArticleRepository constructor(private val remote:ArticleService,private val local:ArticleDao){

    /// 获取文章详情，
    ///
    /// 如果文章以前已被缓存过，那么直接获取缓存的数据，否则拉取服务端的数据并缓存到本地数据库
    /// 返回 [Observable] 给 ViewModel层
    fun getArticleDetail(articleId:String):Observable {
    return local.getLocalArticleById(articleId)
    				 .onErrorResumeNext {
						remote.getArticleById(id)
                        .doOnSuccess { local.insertArticle(it) }
            }
    
}


```

- ##### 推荐参数的转换和返回数据在Model层处理

在进行框架搭建的过程中，我认为能尽量减少错误的方式就是尽可能的让需要调用你方法的人少写代码。

比如以下场景：

```kotlin
/// ViewModel
fun login() {
    final token = "basic"+ base64Encode(utf8.encode('$username:$password'))
    return repository.login(token)
}

/// Model
fun login(token:String){
    return remote.login(token)
}
```

在这种场景下，假如换成了其它的验证方式，那么所有生成token的地方都需要改，耗时耗力，而且如果说有组员生成token的方法错了，那么也挺难排查的。

因此，建议在Model层进行参数的处理

```kotlin
/// ViewModel
fun login() {
    return repository.login(username,password)
}

/// Model
fun login(username:String,password:String){
    final token = "basic"+ base64Encode(utf8.encode('$username:$password'))
    return remote.login(token)
}
```

另外就是**返回数据的转换**。

日常开发中，我们从服务端获取到的数据并不是ViewModel真正需要的，比如会出现`BaseResponse<T>`这样的返回数据，而ViewModel真正需要的则是T。

在前文我们也提到Model层的职责之一便是**提供ViewModel层需要的数据，**因此我们需要在`Repository`中对这类型的数据先处理一番。

```kotlin
/// ViewModel
fun getUserDetail(userId:String) = repository.getUserDetail(userId)

/// Model
fun getUserDetail(userId:String):Observable<User> {
    return remote.getUserDetail(userId)
    .doOnSuccess{
        if(!it.success){
            throw Exception(it.message)
        }
    }
    .map{it.data}
}
//remote
@Get('user/{userId}')
fun getUserDetail(@Path("userId") userId:String):Observable<BaseResponse<User>> 
```

经过这番改造，明确了Model层的职责，我们就可以将重心放在业务逻辑上，写出更加高效的代码。

### 写在最后

关于Model的概念，我想大多数研究或学习过MVX的人都有所了解。但实际应该怎么做，怎么确定Model的职责这个还是看个人的积累。

我也不敢说我的写法就一定是对的，因为我所写的MVVM和其它人包括AAC都有不同的地方，但对于我来说，依循着这样的规范，已经给我包括团队的开发效率带来了极大的提升。

如果你对于文章中的代码有疑问或者感兴趣，可以看看我写的小专栏 [《使用Kotlin构建MVVM应用程序》](https://xiaozhuanlan.com/ditclear)。

如果本文对你有帮助，请点赞支持。

==================== 分割线 ======================

如果你想了解更多关于MVVM、Flutter、响应式编程方面的知识，欢迎关注我。

##### 你可以在以下地方找到我：

简书：https://www.jianshu.com/u/117f1cf0c556

掘金：https://juejin.im/user/582d601d2e958a0069bbe687

Github: https://github.com/ditclear

![](http://upload-images.jianshu.io/upload_images/3722695-812afe12bc7a15fb?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)





