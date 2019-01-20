---
layout: post
title:  "使用Vue快速生成shape背景图"
date:   2019-01-20 10:18:00
categories: Android Vue shape
---

在日常的Android开发之中，我们通常都会根据UI图去手动创建shape或者selector背景图，虽说创建起来很简单，但是未免也会感到繁琐，因此也研究了一些这方面的知识，包括自定义`shapedrawable`、`dataBinding`，到最近看到的通过[LayoutInflater.Factory类](https://juejin.im/post/5b9682ebe51d450e543e3495#comment)等等方法，可见天下真苦秦久矣。

加上最近在学习Vue、小程序等等前端的知识，前面也写过一个为[《MVVM With Kotlin》](https://www.jianshu.com/c/50336d57e9b0)系列提供的脚手架工具.

> generator-mvvm-kotlin : https://github.com/ditclear/generator-mvvm-kotlin

因此便有了一个通过Vue快速生成shape背景图的想法。

### Shape4Android

名称灵感来自于inloop的[shadow4android](http://inloop.github.io/shadow4android/)，Vue看看[官网](https://cn.vuejs.org)的教程，边看边实践，css不熟悉，所以直接搬的[element-ui](http://element-cn.eleme.io/#/zh-CN/component/installation)，还用到了以前收藏的[You-need-to-know-css](https://lhammer.cn/You-need-to-know-css/#/)的技巧，趁着周末完善了一下，上传到了github-pages上。

在线体验：[https://ditclear.github.io/shape4android/](https://ditclear.github.io/shape4android/)

![](https://user-gold-cdn.xitu.io/2019/1/20/1686ad6263d6cb27?w=3474&h=2278&f=png&s=626533)



#### Feature

- 支持常用的retangle和oval两种样式
- 支持设置颜色
- 支持shape和selector (selector支持常用的pressed和unable)

- 支持设置圆角
- 支持设置边框宽度和颜色
- 支持修改文件名称

#### 默认命名规则

shape： `shape_type_color_roundTL_roundTR_roundBL_roundBR_borderWidth_borderColor.xml`  

selector：`selector_shape_n_color_p_pressedColor_u_unableColor.xml`

> 如果自定义文件名称，那么selector中的shape默认会跟上type名，比如xx_norm.xml/xx_pressed.xml/xx_unable.xml

#### TODO

- [ ] 更多形状
- [ ] 虚线
- [ ] 渐变色
- [ ] rippleColor
- [ ] 优化界面

### 写在最后

实质上跟APT编译生成所需的Java/Kotlin文件差不多，但是Vue能够节省很多编译的时间，所以写着感觉很快。

代码实际上也非常简单，就是根据填写的参数拼装成shape或者selector.xml文件而已，毕竟都是模板化的代码，再用filesaver.js下载下来就行。

Github : [https://github.com/ditclear/shape4android](https://github.com/ditclear/shape4android)













