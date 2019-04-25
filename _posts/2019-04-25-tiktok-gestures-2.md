---
layout: post
title:  "使用Flutter仿写TikTok的手势交互效果(二)"
date:   2019-04-25 10:18:00
categories: Flutter
---


Flutter 是 Google推出并开源的移动应用开发框架，主打跨平台、高保真、高性能。开发者可以通过 Dart语言开发 App，一套代码同时运行在 iOS 和 Android平台。

> Flutter官网：[flutter-io.cn](https://link.juejin.im?target=https%3A%2F%2Fflutter-io.cn)

在上一篇手势交互的文章中，我们了解了[GestureDetector](https://flutterchina.club/gestures/)、[Transform](https://book.flutterchina.club/chapter5/transform.html)以及[Hero动画](https://book.flutterchina.club/chapter9/hero.html)，并完成了几个TikTok中的手势交互效果，本文继续前文的内容，如果你再看看上一篇内容，可以查看以下链接：

> 使用Flutter仿写TikTok的手势交互:https://dwz.cn/xoY8eDs0

来看看本次实现的效果：

![](https://media.giphy.com/media/hVaGnlmHEUt5rh08mB/giphy.gif)



> Gif：https://media.giphy.com/media/hVaGnlmHEUt5rh08mB/giphy.gif
>
> Github地址：https://github.com/ditclear/tiktok_gestures

#### 下载体验

![](https://user-gold-cdn.xitu.io/2019/4/25/16a53d44dcc75d3a?w=300&h=300&f=png&s=10784)

### 交互分解

本次主要包含两个下拉的交互：下拉刷新和下拉返回。

- 下拉刷新

![](https://media.giphy.com/media/UUmpalQGhGmd6EEpWh/giphy.gif)

> Gif：https://media.giphy.com/media/UUmpalQGhGmd6EEpWh/giphy.gif

随着手指的向下滑动，offsetY的改变，这里有两个变化：

1. 透明度
2. y轴方向的偏移

透明度这里可以采用Flutter提供的Opacity部件，这是专门用来进行透明度变化的，用法也很简单。

```dart
Opacity(
  opacity: currentOpacity,
  child:childWidget
)  
```

传入opacity即可，我们要做的也就是将opacity的值用offsetY表示出来。

值得注意的是这里有两个透明度的变化:**顶部导航栏**和**下拉刷新内容的文本**。

而且二者不会同时出现，当下拉时，**顶部导航栏**逐渐变为透明，到一定距离时(可以认为是最大下拉距离的一半)就隐藏，**下拉刷新内容的文本**才会出现，而且其透明度逐渐变为不透明。

我们把下拉的最大滑动距离设为40，那么这个临界点就是20。

由此可以得出**顶部导航栏**随着下拉距离透明度变化的公式。

> opacity = 1 - offsetY / 20

因为offsetY最大可能为40，而opacity必须在0到1之间。因此最后得出的公式是：

> opacity = max(0, 1 - offsetY / 20)

具体实现：

![](https://upload-images.jianshu.io/upload_images/3722695-9ec69c7c1cf2b13c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

当下拉的时候，记录下偏移总量`offsetY`，控制其大于0并且不超过最大距离即可。

```dart
onVerticalDragUpdate: (details) {
        final tempY = offsetY + details.delta.dy / 2;
        if (currentIndex == 0) {
          //最大下拉距离不超过40
          if (tempY > 0) {
            if (tempY < 40) {
              setState(() {
                offsetY = tempY;
              });
            } else if (offsetY != 40) {
              setState(() {
                setState(() {
                  offsetY = 40;
                });
              });
              // 当下拉到最大距离时，触发震动效果
              vibrate();
            }
          }
        } else {
          offsetY = 0;
        }
      }

```

当下拉到最大距离时，可以使用[vibrate](https://pub.dartlang.org/packages/vibrate)来产生震动效果。

当手指离开屏幕时，再进行一个动画，将OffsetY设置为0，并通过setState通知UI重新渲染。

```dart
// 下拉结束
onVerticalDragEnd: (_) {
    if (offsetY != 0) {
       animateToTop();
     }
}

/// 滑动到顶部
///
/// [offsetY] to 0.0
void animateToTop() {
  animationControllerY =
      AnimationController(duration: Duration(milliseconds: offsetY.abs() * 1000 ~/ 40), vsync: this);
  final curve = CurvedAnimation(parent: animationControllerY, curve: Curves.easeOutCubic);
  animationY = Tween(begin: offsetY, end: 0.0).animate(curve)
    ..addListener(() {
      setState(() {
        offsetY = animationY.value;
      });
    });
  animationControllerY.forward();
}
```

- 下拉返回

![](https://media.giphy.com/media/KCq45wtPB0vsJzfYmT/giphy.gif)

> Gif:https://media.giphy.com/media/KCq45wtPB0vsJzfYmT/giphy.gif

简单的讲就是下拉的时候，对整个页面进行一个Y轴方向的偏移，当超过一个指定的距离的时候，退出这个页面即可。

```dart
// 滑动截止时
onPanEnd: (_) {
  if (offsetY > 100) {
    // 下拉距离超过100，即退出页面
    Navigator.pop(context);
  } else if (offsetY > 0) {
    // 下拉距离小于100，恢复原样
    animateToBottom(screenHeight);
  } else if (offsetY < 0) {
    // 上拉根据是否已经显示评论框 [isCommentShow]和offsetY来判断是展开还是收缩
    if (!isCommentShow && offsetY.abs() > screenHeight * 0.2) {
      if (offsetY.abs() > screenHeight * 0.2) {
        animateToTop(screenHeight);
      } else {
        animateToBottom(screenHeight);
      }
    } else {
      if (offsetY.abs() > screenHeight * 0.4) {
        animateToTop(screenHeight);
      } else {
        animateToBottom(screenHeight);
      }
    }
  }
},
```

页面偏移的话，在最外层套一层`Transform.translate`，注意偏移量需要大于0。

```dart
Transform.translate(
  offset: Offset(0, max(0, offsetY)),
  child: childWidget
  ）
```

当你完成了上诉的基本逻辑时，运行之后会发现跟预想的还是有些出入。

![](https://upload-images.jianshu.io/upload_images/3722695-bc90c174c921b0b8.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/240)

背景不透明，当时我认为是有一个黑色的背景，需要设置背景色的透明度，可是在ThemeData里的各种可疑的颜色都试过，发现并没有什么用，后来想到应该是PageRoute的原因。

在我们调用`Navigator`进行push操作的时候，都需要传入一个`PageRoute`，比如说常用的`MaterialPageRoute`或者`CupertinoPageRoute`，在`PageRoute`里有一个`opaque`，不透明的意思，而且一直为true。

```
@override
bool get opaque => true;
```

为了解决上述问题，这里copy了一份`MaterialPageRoute`的源码然后将`opaque`改为false就可以了。

```dart
/// copy 一份 MaterialPageRoute，修改opaque
class TransparentPage<T> extends PageRoute<T> {
  //...
  /// false 代表背景透明
  @override
  bool get opaque => false;
   
  //...  
}
```

最后，在上一篇文章中有同学问如何在列表中进行Hero动画，在代码中也模拟了一下，保证Hero的tag相同即可，Flutter会在新旧路由切换的时候，对相同tag的Hero部件进行动画。

```dart
/// detail_page.dart
child: Hero(
  tag: "detail_$currentIndex",
  child: GestureDetector(
      child:PageView(
                onPageChanged: (index) {
                  setState(() {
                    currentIndex = index;
                  });
                },
          //..
         ),
      ),
    )
    
/// right_page.dart
child: Hero(
      tag: "detail_0",
      child: Image.asset(
             assets/detail.png",
             fit: BoxFit.fill,
       ),
   ),
```

### 写在最后

有了手势交互的经验，完成上述的效果还是不难的。

稍微费点时间就是背景色的问题了，不过Flutter是开源的，而且注释和用例都写得非常详细，如果说Flutter框架不能满足你的需求，那么完全可以修改Flutter底层的源码来达到想要的效果。

到此，本篇的内容也结束了，最后还剩下一个手势冲突的处理，这个内容就放在下一篇吧，关注我获取最新文章哦。

> Github地址：https://github.com/ditclear/tiktok_gesturess

























