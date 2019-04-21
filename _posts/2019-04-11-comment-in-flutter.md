---
layout: post
title:  "Effective Dart 文档注释在Flutter项目中的实践"
date:   2019-04-11 10:18:00
categories: Dart Flutter
---

什么是注释？

> 在编程语言中，注释就是对代码的解释和说明，其目的是让人们能够更加轻松地了解代码。

也有一句话是这样说的：**程序员都讨厌两件事，1.别人不写注释 2.自己写注释**

在开发者社区里，我不止一次的看到吐槽离职的前同事不写注释的例子，其实不光是他人的代码，即使是自己写的代码，一段时间以后再去看，你也会发现：**这写的什么呀**？

作为开发者，我们大多都知道编写注释的重要性，但是却往往抱着"**能实现功能就可以了**"这样的心态去写代码，完全随心所欲的去实现，结果当然就是留下一堆烂摊子，形成前文所述的恶性循环，所以在编写代码的同时，请牢记**代码首先是写给人看的**。

**编写精炼的、准确的注释 只需要几秒钟，但是以后可能节省其他人几个小时 的时间来读懂您的代码。**

### 写好注释

写注释简单吗？简单。那么写好注释，简单吗？

答案就跟写文章一样，有的文章是记流水账，有的则是发人深省，回味无穷。

注释也是同样，在[《代码精进之路》](https://time.geekbang.org/column/intro/129)里总结了编写注释的三项原则：

1. **准确**，错误的注释比没有注释更糟糕。
2. **必要**，多余的注释浪费阅读者的时间。
3. **清晰**，混乱的注释会把代码搞得更乱。

比如，当我们说编程语言时，一定不要省略“编程”这两个字。否则，就可能被误解为大家日常说话用的自然语言。这就是准确性的要求。

**bad:**

```dart
String language = "Java"; // the language
```

**better**:

```dart
String language = "Java"; // the programming language
```

如果代码已经能够清晰、简单地表达自己的语义和逻辑，这时候重复代码语义的注释就是多余的注释。注释的维护是耗费时间和精力的，所以，不要保留多余的、不必要的注释。还有一句很精辟的话：**Code tells you How, Comments tell you Why.**

**bad:**

```dart
// the programming language
String programmingLanguage = "Java";
```

**better:**

```dart
String programmingLanguage = "Java";
```

如果注释和代码不能从视觉上清晰地分割，注释就会破坏代码的可读性。

**bad:**

```dart
/* dump debug information
if (hasDebug) {
  System.out.println("Programming language: Jave");
} */
String programmingLanguage = "Java";
```

**better:**

```dart
// dump debug information
//
// if (hasDebug) {
//   System.out.println("Programming language: Jave");
// }
String programmingLanguage = "Java";
```

[《Effective Dart》](http://dart.goodev.org/guides/language/effective-dart/documentation#section-1)中关于如何写注释，也推荐要**清晰和准确，同时还有简洁**。

在运用Dart语言编写Flutter应用的过程中，由于视图层都是纯Dart代码，而且夹杂着许多的`if..else..`，需要根据条件显示不同的widget，因此在做好widget拆分的同时，编写良好的注释势在必行。

### Effective Dart 文档注释

在学习并使用Flutter框架开发App的过程中，有些开发者并没有怎么关注Dart语言，草草看了几下语法之后就开始了Flutter之旅，还是按照以前的Java、Swift这样的语法风格进行开发，这在以后可能会带来不必要的麻烦，比如团队成员里各个都不同的命名方式、注释也各不相同等等。

因此推荐先看一下 [《Effective Dart》](http://dart.goodev.org/guides/language/effective-dart)，主要内容包含**代码风格**、**文档注释**、**最佳实践**、**设计指南**，能从中获益良多。我们主要关心[**文档注释**](http://dart.goodev.org/guides/language/effective-dart/documentation)这一章节，这里我总结了以下两点：

#### 1.使用`///`放弃`/** ... */`

在[《Effective Dart》](http://dart.goodev.org/guides/language/effective-dart/documentation#section-1)解释了相应原因

> 由于历史原因，dartdoc 支持两种格式的文档注释： `///` (“C# 格式”) 和 `/** ... */` (“JavaDoc 格式”)。我们推荐使用 `///` 是因为其更加简洁。`/**` 和 `*/` 在多行注释中间添加了开头和结尾的两行多余 内容。 `///` 在一些情况下也更加易于阅读，例如 当注释文档中包含有使用 `*` 标记的列表内容的时候。
>
> 如果你现在还在使用 JavaDoc 风格格式，请考虑 使用新的格式。

与此同时，要使用 `///` 文档注释来注释成员和类型。

**bad：**

```dart
// The number of characters in this chunk when unsplit.
int get length => ...
```

**better:**

```dart
/// The number of characters in this chunk when unsplit.
int get length => ..
```

而`//`则主要用于方法体内的注释

```dart
greet(name) {
  // Assume we have a valid name.
  print('Hi, $name!');
}
```

#### 2. 把第一个语句定义为一个段落并使用散文的方式来描述

注释文档中的第一个段落应该是简洁的、面向用户的注释。例如下面的示例， 通常不是一个完成的语句。

**bad:**

```dart
/// Starts a new block as a child of the current chunk. Nested blocks are
/// handled using their own independent [LineWriter].
ChunkBuilder startBlock() { ... }
```

**better：**

```dart
/// Defines a flag.
///
/// Throws an [ArgumentError] if there is already an option named [name] or
/// there is already an option using abbreviation [abbr]. Returns the new flag.
Flag addFlag(String name, String abbr) { ... }
```

这就跟日常写博客一样，会先写一个段落大意，然后围绕着这点进行详细描述，这样更容易让读者理解核心思想，也更加节省时间。

另外**推荐使用散文的方式来描述参数、返回值以及异常信息。**

在其他语言中，比如`JavaDoc`使用各种标签和额外的注释来描述参数和 返回值。

**bad:**

```dart
/// Defines a flag with the given name and abbreviation.
///
/// @param name The name of the flag.
/// @param abbr The abbreviation for the flag.
/// @returns The new flag.
/// @throws ArgumentError If there is already an option with
///     the given name or abbreviation.
Flag addFlag(String name, String abbr) { ... }
```

而 Dart 把参数、返回值等描述放到文档注释中，并使用方括号来引用 以及高亮这些参数和返回值。 

**better:**

```dart
/// Defines a flag.
///
/// Throws an [ArgumentError] if there is already an option named [name] or
/// there is already an option using abbreviation [abbr]. Returns the new flag.
Flag addFlag(String name, String abbr) { ... }
```

我们主要注意这两点，其它规范可以查看[《文档注释》](http://dart.goodev.org/guides/language/effective-dart/documentation#section-1)。

### 在mvvm_flutter项目中的实践

[mvvm_flutter](https://github.com/ditclear/mvvm_flutter)是我在Flutter中运用MVVM架构的一个示例。

> mvvm_flutter : https://github.com/ditclear/mvvm_flutter

项目完成后只简单的在几个关键方法上添加了几个`JavaDoc`格式的注释，比如：

```dart
 /**
   * call the model layer 's method to login
   * doOnData : handle response when success
   * doOnError : handle error when failure
   * doOnListen ： show loading when listen start
   * doOnDone ： hide loading when complete
   */
  Observable login() => _repo
      .login(username, password)
      .doOnData((r) => response = r.toString())
      .doOnError((e, stacktrace) {
        if (e is DioError) {
          response = e.response.data.toString();
        }
      })
      .doOnListen(() => loading = true)
      .doOnDone(() => loading = false);
```

现在我们开始将其转换为Dart语言推荐的注释风格。如下所示：

```dart
  /// 登录
  ///
  /// 调用 model层[GithubRepo] 的 login 方法进行登录
  /// 传入 [username] 和 [password] 
  /// 成功：显示返回的信息
  /// 失败：处理错误，显示错误信息
  /// 订阅开始：loading = true
  /// 订阅结束：loading = false
  /// 返回 [Observable] 给 View 层
  Observable login() => _repo
      .login(username, password)
      .doOnData((r) => response = r.toString())
      .doOnError((e, stacktrace) {
        if (e is DioError) {
          response = e.response.data.toString();
        }
      })
      .doOnListen(() => loading = true)
      .doOnDone(() => loading = false);
```

相比之前的代码，清晰干净了蛮多，我们在注释的第一行中描述了这个方法的功能，随后以散文的方式对方法的调用、参数以及返回值进行了描述。

这是ViewModel层的注释，接下来我们来进行Model层的注释。

```dart
class GithubRepo {
    /// ...
  Observable login(String username, String password) {
    _sp.putString(KEY_TOKEN, "basic " + base64Encode(utf8.encode('$username:$password')));
    return _remote.login();
  }
}

```

我们对`login`方法进行注释，结果如下：

```dart
/// 仓库层
class GithubRepo {
	
  /// 登录
  ///
  /// 将ViewModel层 传递过来的[username] 和 [password] 处理为 token 并用[_sp]进行缓存
  /// 调用 [_remote] 的 [login] 方法进行网络访问
  /// 返回 [Observable] 给ViewModel层
  Observable login(String username, String password) {
    _sp.putString(KEY_TOKEN, "basic " + base64Encode(utf8.encode('$username:$password')));
    return _remote.login();
  }
}

```

这里方法比较简单，但对于复杂的仓库层的方法，可能包含着网络、数据库、`MethodChannel`、`SharedPreferences`等等交互，运用Dart推荐的注释方式，可以更好的描述你的代码逻辑。

```dart
  /// 获取文章详情
  ///
  /// 1.先通过 数据库[_local] 查看本地是否有 id 为 [articleId]的文章
  /// 2.有缓存则到第4步，没有缓存则到第3步
  /// 3.通过网络层[_remote] 获取服务端数据，成功后再进行缓存 ,到第4步
  /// 4.返回 [Observable] 给ViewModel层
  Observable getArticleDetail(int articleId) {
    return _local
        .getArticleById(articleId)
        .onErrorResumeNext(
        _remote.getArticleById(articleId)
            .doOnData((article) => _local.insertArticle(article)));
  }
```

这样，我们轻松的拆解了代码逻辑，也能更容易的编写出相应的测试用例，方便进行单元测试。

而`View`层相比`Model`层和`ViewModel`层，需要额外注意的是其夹杂着逻辑判断，需要根据条件显示不同的`Widget`，我们在将复杂部分提取成方法的时候，也需要对其进行详细的描述。

```dart
/// 登录按钮内部的widget
///
/// 当请求进行时 [value.loading] 为 true 时,显示 [CircularProgressIndicator]
/// 否则显示普通的登录文本
Widget buildLoginChild(HomeProvide value) {
  if (value.loading) {
    return const CircularProgressIndicator();
  } else {
    return const FittedBox(
      fit: BoxFit.scaleDown,
      child: const Text(
        'Login With Github Account',
        maxLines: 1,
        textAlign: TextAlign.center,
        overflow: TextOverflow.fade,
        style: const TextStyle(fontWeight: FontWeight.bold, fontSize: 16.0, color: Colors.white),
      ),
    );
  }
}
```

经过这番改进，当其他开发者阅读您的代码时，也能减少很多不必要的烦恼。

### 写在最后

在运用[《Effective Dart》](http://dart.goodev.org/guides/language/effective-dart/documentation#section-1)中的注释技巧进行文档注释时，会有一种在写博客的感觉，因为写博客的好处之一便是**备忘**，将来再复习的时候能够更加快速的理解知识点，注释也是这样。

我认为即使是再优秀的开发者，如果不写注释，时间长了，也会忘记，而且有幸见识过一个Java文件包含了30000+行代码，还没有注释，到现在都没有人愿意去接手这样的项目，大家都是程序员，程序员何苦为难程序员呢。

#### 参考资料：

[Code Complete-自说明代码](https://book.douban.com/subject/1477390/?i=0)

[代码精进之路-写好注释，真的是小菜一碟？](https://time.geekbang.org/column/intro/129)

[Effective Dart - 文档注释](http://dart.goodev.org/guides/language/effective-dart/documentation#section-1)

