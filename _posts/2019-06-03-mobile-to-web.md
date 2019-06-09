---
layout: post
title:  "Flutter全平台！迁移现有Flutter项目到WEB端"
date:   2019-06-03 10:18:00
categories: Flutter
---


Flutter 是 Google推出并开源的移动应用开发框架，主打跨平台、高保真、高性能。开发者可以通过 Dart语言开发 App，一套代码同时运行在 iOS 、Android、web和桌面端。

> Flutter官网：[https://flutter-io.cn](https://flutter-io.cn/)

[Flutter_web](https://github.com/flutter/flutter_web)是Flutter代码兼容web的实现，可以将使用Dart编写的现有Flutter代码编译成可以嵌入浏览器并部署到任何Web服务器的客户端。

> Our goal is to enable building applications for mobile and web simultaneously from a single codebase. However, to allow experimentation, the tech preview Flutter for web is developed in a separate namespace. So, as of today an existing mobile Flutter application will not run on the web without changes.
>
> Flutter的目标是通过单一代码库同时构建移动和Web应用程序。 但是，为了进行实验，Flutter_web是在一个单独的命名空间中开发的。 因此，截至目前，现有的移动Flutter应用程序无法在不进行更改的情况下在Web上运行。

简而言之就是Flutter现在还不支持既是移动应用也是Web应用，需要自己进行迁移，但相信日子不会太远。

### 迁移Flutter项目到WEB端

这次我们迁移的项目是[flutter_challenge_googlemaps](https://github.com/flutter-ui-challenges/flutter_challenge_googlemaps)，效果图如下：

![](https://upload-images.jianshu.io/upload_images/3722695-df03a3527c62a53d?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

#### 怎么做？

大多数Dart代码都是共用的，需要改变的只是一些依赖和配置。

首先是`pubspec.yaml`需要用flutter_web来替换flutter，同时移除asset和字体相关的代码。

```dart
name: flutter_web_challenge_googlemaps

environment:
  # You must be using Flutter >=1.5.0 or Dart >=2.3.0
  sdk: '>=2.3.0-dev.0.1 <3.0.0'

dependencies:
  flutter_web: any
  flutter_web_ui: any


dev_dependencies:
  build_runner: ^1.4.0
  build_web_compilers: ^2.0.0

dependency_overrides:
  flutter_web:
    git:
      url: https://github.com/flutter/flutter_web
      path: packages/flutter_web
  flutter_web_ui:
    git:
      url: https://github.com/flutter/flutter_web
      path: packages/flutter_web_ui
  flutter_web_test:
    git:
      url: https://github.com/flutter/flutter_web
      path: packages/flutter_web_test
```

通过`flutter package get`更新依赖后，需要更新`lib`路径下dart文件中的相关引用。

```dart
//flutter
import 'package:flutter/material.dart';
//flutter web
import 'package:flutter_web/material.dart';
```

差别就是将`flutter`替换为`flutter_web`而已，代码基本不用动。

接下来，为了预览网页，我们需要自己创建web目录，并在目录下创建`web/index.html` 和 `web/main.dart`文件。

**web/index.html**

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <title></title>
  <script defer src="main.dart.js" type="application/javascript"></script>
</head>
<body>
</body>
</html>
```

**web/main.dart**

```dart
import 'package:flutter_web_ui/ui.dart' as ui;
// 将flutter_web_challenge_googlemaps替换为自己的package
import 'package:flutter_web_challenge_googlemaps/main.dart' as app;

main() async {
  await ui.webOnlyInitializePlatform();
  app.main();
}

```

至于资源文件、图片、字体等，和Flutter项目不同，这些都需要放到`web\assets`目录路径下，同时要记得更新代码中的相关引用。

```dart

Image.asset("assets/logo.ong");
// 需要更改为
Image.asset("logo.png");

```

如果你有使用Material Icon的话，你需要在`web/assets`目录下创建`FontManifest.json`文件，并添加相关地址。

```json
[
  {
    "family": "MaterialIcons",
    "fonts": [
      {
        "asset": "https://fonts.gstatic.com/s/materialicons/v42/flUhRq6tzZclQEJ-Vdg-IuiaDsNcIhQ8tQ.woff2"
      }
    ]
  }
 
]

```

整个web目录会如下图所示:

![web](https://upload-images.jianshu.io/upload_images/3722695-e53af89ae167b6f3?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

运行项目，可以发现和移动端基本没有区别。

![image](https://upload-images.jianshu.io/upload_images/3722695-fc67deb510850c97?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

效果还是蛮流畅的🤙。

![](https://upload-images.jianshu.io/upload_images/3722695-2de5a94e74a41f49?imageMogr2/auto-orient/strip)
如果你想查看release版本，可以运行

> flutter pub global run webdev serve -r

如果你想发布制品，则可以运行

> flutter pub global run webdev build

这会在项目的根路径下生成一个build文件夹，里面包含可以部署到服务器上的文件，如下图所示：
![](https://upload-images.jianshu.io/upload_images/3722695-d997801ef5ccbc56?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
你可以运用gh-pages简单的将其部署到Github上，然后预览效果

[https://flutter-ui-challenges.github.io/flutter_web_challenge_googlemaps/#/](https://flutter-ui-challenges.github.io/flutter_web_challenge_googlemaps/#/)

> 关于如何运用gh-pages进行页面预览，可以查看此链接：[https://www.cnblogs.com/MuYunyun/p/6082359.html](https://www.cnblogs.com/MuYunyun/p/6082359.html)

### 写在最后

虽然说跨平台的项目很多的，比如weex、RN、Kotlin等等，但是真正让我体会到跨平台高效一体的体验还是Flutter，这也许就是为什么年后我一直在学习和从事Flutter开发的原因之一了。

当然flutter_web还处于早期阶段，一些flutter的功能还没有完全移植过来，比如高斯模糊效果，不过Flutter1.0正式版本才到来不久，相信在不久的将来，这些全都会有。

#### 参考文档

flutter_web：<https://github.com/flutter/flutter_web>

迁移指南：<https://github.com/flutter/flutter_web/blob/master/docs/migration_guide.md>


