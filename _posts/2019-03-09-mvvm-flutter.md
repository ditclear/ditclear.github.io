---
layout: post
title:  "MVVM架构在Flutter中的简单实践"
date:   2019-03-09 10:18:00
categories: MVVM Flutter
---

Flutter 是 Google推出并开源的移动应用开发框架，主打跨平台、高保真、高性能。开发者可以通过 Dart语言开发 App，一套代码同时运行在 iOS 和 Android平台。

> Flutter官网：https://flutter-io.cn

还记得18年参加上海Google开发者大会的时候，听了一天的Flutter的介绍，之后不久1.0发布了，到现在1.2版本，Flutter一直都在快速的进步着。年后我也终于开始了Flutter的深入学习，并很快有机会直接在项目中进行实践。

如果你也是刚开始学习Flutter，推荐以下资源：

- IDE：Android Studio ，相比VS Code 具备更多的debug工具，更好的代码提示和跳转，毕竟都是Google自家的东西。
- 起步：[编写你的第一个 Flutter App](https://codelabs.flutter-io.cn/codelabs/first-flutter-app-pt1-cn/index.html#0)
- 入门：[Flutter In Action](https://book.flutterchina.club/)
- 状态管理：ScopeModel、redux、Bloc、Provide。可以查看[Vadaski老哥的文章](https://juejin.im/user/5b5d45f4e51d453526175c06/posts)。
- 实践：重构现有项目的一个页面，并尝试集成到原生项目中。
- 深入：查看源码并知晓Flutter运行原理。

### MVVM-Flutter

项目架构当然是MVVM，依旧遵循[App开发架构指南](https://link.jianshu.com/?t=http://www.jcodecraeer.com/a/anzhuokaifa/androidkaifa/2017/0523/7963.html)。对比以前写的[MVVM-Android](https://github.com/ditclear/MVVM-Android)，发现有许多的共通之处，将依赖替换成Flutter版本之后就是熟悉的节奏。

**项目地址**：https://github.com/ditclear/mvvm_flutter

![MVVM](https://user-gold-cdn.xitu.io/2019/3/9/16961e49beffae75?w=1306&h=1000&f=png&s=73739)

#### dependencies

- [dio](https://github.com/flutterchina/dio) : 网络请求
- [rxdart](https://github.com/ReactiveX/rxdart)：响应式编程
- [flutter-provide](https://github.com/google/flutter-provide)：通知ui更新数据

> 思想：M-V-VM各层直接通过rx衔接，配合响应式的思想和rxdart的操作符进行逻辑处理，最后通过provide来更新视图。

#### Code

```dart
//remote
class GithubService{
  Observable<dynamic> login()=> get("user");
}
//repo
class GithubRepo {
  final GithubService _remote;

  GithubRepo(this._remote);

  Observable login(String username, String password) {
    token = "basic " + base64Encode(utf8.encode('$username:$password'));
    return _remote.login();
  }
}
//viewmodel
class HomeViewModel extends ChangeNotifier {
  final GithubRepo _repo; //数据仓库
  String username = ""; //账号
  String password = ""; //密码
  bool _loading = false; // 加载中
  String _response = ""; //响应数据
  //...
  HomeViewModel(this._repo);

  /**
   * 调用model层的方法进行登录
   * doOnData : 请求成功时，处理响应数据
   * doOnError : 请求失败时，处理错误
   * doOnListen ： 开始时loading为true,通知ui更新
   * doOnDone ： 结束时loading为false,通知ui更新
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
}

//view
class HomeWidget extends StatefulWidget {
  @override
  State<StatefulWidget> createState() {
    return _HomeState(provideHomeViewModel());
  }
}
class _HomeState extends State<HomeWidget>{
   //...
  _HomeState(this._viewModel) {
    providers.provideValue(_viewModel);
  }
	
  @override
  Widget build(BuildContext context) {
    return Scaffold(
        appbar://...,
        body://...
       
        CupertinoButton(
            onPressed: _login,
            //...
         ),
         Container(
                //...
                child: Provide<HomeViewModel>(
                  builder: (BuildContext context, Widget child,
                          HomeViewModel value) =>
                      Text(value.response),
                ),
              ),
        //...
        
        );
  }
  
    
  _login()=>_viewModel.login().doOnListen(() {
      _controller.forward();
    }).doOnDone(() {
      _controller.reverse();
    }).listen((_) {
      //success
      Toast.show("login success",context,type: Toast.SUCCESS);
    }, onError: (e) {
      //error
      dispatchFailure(context, e);
    });
 
}
```

最后效果：

[![screeshot](https://github.com/ditclear/mvvm_flutter/raw/master/screenshot.png)](https://github.com/ditclear/mvvm_flutter/blob/master/screenshot.png)

#### 下载体验

[![Android](https://github.com/ditclear/mvvm_flutter/raw/master/android.png)](https://github.com/ditclear/mvvm_flutter/blob/master/android.png)

### 写在最后

Flutter上手还是蛮容易的，大概一周的时间就能熟悉，毕竟很多东西Flutter团队都帮你弄好了。

而且hot reload真的舒服，一次编写，两端运行。当然缺点也是有的，比如：

- 插件质量较低（毕竟才起步，相信以后会完善）
- json解析真的烦（远不如kotlin简单,而且没有扩展）
- 布局层次较多（配上flutter dev tool ，在可接受范围内）

总的来说，**我愿意在Flutter上投入时间，相信也能得到相应的回报**，毕竟都9102年了，是吧?

如果你想了解更多关于MVVM、Flutter、响应式编程方面的知识，欢迎关注我。


