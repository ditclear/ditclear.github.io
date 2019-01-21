---
layout: post
title:  "使用Kotlin构建MVVM应用程序—完结篇：快速开发"
date:   2019-01-19 10:18:00
categories: Android Kotlin MVVM
---

大家好，这里是使用Kotlin构建MVVM应用程序—完结篇。

这个系列断断续续的写了一年之久，期间我也在不断验证和完善所学和所想，终于在18年的末尾算是收尾了。

在这一年多的时间里，很感谢大家的支持和认可，希望[《使用Kotlin构建MVVM应用程序》](https://xiaozhuanlan.com/ditclear)这一系列文章能够帮助更多的Android开发者过渡到MVVM架构和Kotlin语言，结合响应式的思想，能够让开发者的重心放到更值得做的事情上。

你可以查看以下链接学习如何使用Kotlin构建MVVM应用程序

> 简书专题：<https://www.jianshu.com/c/50336d57e9b0>

> 小专栏：<https://xiaozhuanlan.com/ditclear?rel=2493325177>

趁着今天是圣诞节，我写了两个库帮助学习和使用本系列的同学，更快速的进行MVVM形式的开发。

### MVVM-Kotlin 脚手架工具

github地址：https://github.com/ditclear/generator-mvvm-kotlin

简介：为[《MVVM With Kotin》](https://www.jianshu.com/c/50336d57e9b0) 系列 打造的脚手架工具，提供CLI支持，免去重复创建Android工程的烦恼。

#### Installation

首先，使用npm安装 [Yeoman](http://yeoman.io/)和[generator-mvvm-kotlin](https://www.npmjs.com/package/generator-mvvm-kotlin)

```bash
npm install -g yo
npm install -g generator-mvvm-kotlin
```

然后便可以快速搭建MVVM-Kotlin项目：

```bash
mkdir NewApp
cd NewApp
yo mvvm-kotlin
```

![](https://user-gold-cdn.xitu.io/2018/12/25/167e497db0d2d189?w=1644&h=1082&f=png&s=520515)

使用Android Studio打开NewApp

![](https://user-gold-cdn.xitu.io/2018/12/25/167e4b09886e7c4f?w=1240&h=813&f=png&s=220274)

### AAMVVM 模板代码

github地址：https://github.com/HeadingMobile/AAMVVM

简介： 快速开发Android MVVM应用程序模板，帮助快速生成ViewModel、View、XML文件

![](https://user-gold-cdn.xitu.io/2018/12/25/167e4aef9541bf53?w=1240&h=813&f=png&s=292470)

提供

- MVVMActivity
- MVVMFramgnet

#### Installation

- MAC

打开终端terminal

```
cd /Applications/Android\ Studio.app/Contents/plugins/android/lib/templates
git clone https://github.com/HeadingMobile/AAMVVM.git
```

- Windows

打开终端cmd

```
cd ${Android studio路径}\plugins\android\lib\templates
// 例：cd C:\Program Files\Android\Android Studio\plugins\android\lib\templates
git clone https://github.com/HeadingMobile/AAMVVM.git
```

然后重启Android Studio。

在对应的目录下右击，选择所需的MVVM模板，提供Java 和 Kotlin版本。

> 注意：依赖注入默认使用Koin，配套MVVM-Kotlin使用，基类请参考[PaoNet](https://github.com/ditclear/PaoNet)示例代码。

#### 字段说明

![](https://user-gold-cdn.xitu.io/2018/12/25/167e4aef9521fb5f?w=1240&h=1066&f=png&s=251641)

| 字段              | 说明                                                         |
| ----------------- | ------------------------------------------------------------ |
| Short Name        | 页面功能简称                                                 |
| generateViewModel | 是否生成ViewModel，默认生成                                  |
| Package name      | 该页面的packageName                                          |
| Module Name       | 默认为app，如果不是位于app模块，请填写名称                   |
| Custom SrcDir     | 默认为src.main.java，如果不是这个路径，比如在src.main.kotlin，请修改 |
| Source Language   | 支持Kotlin、Java语言，Java语言需要开发者实现获取ViewModel方法 |

### 写在最后

虽然《使用Kotlin构建MVVM应用程序系列》已经完结，但我还有蛮多想要实践的地方没来的及学习和验证，包括模块化、组件化、跨平台、kotlin后端学习、Flutter、前端等等的知识，也想分享一些自己认为比较不错的Medium上的文章和开源库。

因此我开通了一个公众号，也叫ditclear，感谢关注！

![](https://user-gold-cdn.xitu.io/2018/12/25/167e4aef95784326?w=1240&h=354&f=png&s=96067)




