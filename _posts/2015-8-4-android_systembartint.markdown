---
layout: post
title:  "Android 修改系统栏颜色"
date:   2015-8-4
---

<p class="intro"><span class="dropcap">相</span>信大家在用其它APP的时候经常看到操作栏的颜色和系统栏的颜色相同,感觉十分漂亮.如:</p>
<div  align="center">  
 <img style="box-shadow:5px 0 5px;" src="{{ site.url }}/assets/img/sample.jpg" alt="sample">
</div>
##那么我们需要怎么做呢？

当然，首先我们需要有ActionBar,添加非常简单，只需要在AndroidManifest.xml中指定Application或Activity的theme是Theme.Holo或其子类就可以了.

在Android4.4以后提供了一套能透明的系统ui样式给状态栏和导航栏,可以很轻松的实现这样的功能。

首先要打开activity的透明主题功能，可以把activity的主题设置继承*.TranslucentDecor 主题，然后设置android:windowTranslucentNavigation 或者android:windowTranslucentStatus的主题属性为true，又或者在activity的代码里面开启FLAG_TRANSLUCENT_NAVIGATION 或是 FLAG_TRANSLUCENT_STATUS的window窗口标识。由于透明主题不能在4.4以前的版本里面使用，所以在4.4以前的版本中系统样式跟以前没有区别。

但幸运的是有大神为我们进行了兼容，可以支持到api 10.

Eclipse需要添加jar包：[地址](https://repository.sonatype.org/service/local/repositories/central-proxy/content/com/readystatesoftware/systembartint/systembartint/1.0.4/systembartint-1.0.4.jar)

Android Studio在app的build.gradle中添加：

```java
dependencies {
    compile 'com.readystatesoftware.systembartint:systembartint:1.0.3'
}
```
然后在activity只需要简单的配置：

```java
 	@Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.KITKAT) {
            setTranslucentStatus(true);
        }

        SystemBarTintManager tintManager = new SystemBarTintManager(this);
        tintManager.setStatusBarTintEnabled(true);
        tintManager.setStatusBarTintResource(R.color.pink);
    }
    @TargetApi(19)
    private void setTranslucentStatus(boolean on) {
        Window win = getWindow();
        WindowManager.LayoutParams winParams = win.getAttributes();
        final int bits = WindowManager.LayoutParams.FLAG_TRANSLUCENT_STATUS;
        if (on) {
            winParams.flags |= bits;
        } else {
            winParams.flags &= ~bits;
        }
        win.setAttributes(winParams);
    }
```

当然还有其它的效果，如系统栏透明和设置其它颜色，具体可见github开源库:[https://github.com/jgilfelt/SystemBarTint](https://github.com/jgilfelt/SystemBarTint)
