---
layout: post
title:  "你可能不知道的DataBinding技巧"
date:   2019-05-25 10:18:00
categories: databinding
---


近期，在笔者开源的[**BindingListAdapter**](https://github.com/ditclear/BindingListAdapter)库中出现了这样的一个Issue。

![](http://upload-images.jianshu.io/upload_images/3722695-e85710822cf8a1d9?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

这其实是一个在列表中比较常见的问题，在没有找到比较好的解决办法之前，确实都是通过整项刷新`notifyDataChanged`来保证数据显示的正确性。到后来的`notifyItemChanged`和更佳的`DiffUtil`，说明开发者们一直都在想办法来解决并优化它。

但其实如果你使用DataBinding，做这个局部刷新或者说是定点刷新，就很简单了，这可能是大多数使用DataBinding的开发者并不知道的技巧。

### 巧用ObservableFiled

可以先看看实际的效果

![](http://upload-images.jianshu.io/upload_images/3722695-60a2ecacf653bba8?imageMogr2/auto-orient/strip)


关键点就在于不要直接绑定具体的值到xml中，应先使用`ObservableField`包裹一层。

```kotlin
class ItemViewModel constructor( val data:String){
    //not
    //val count = data
    //val liked = false
    
    //should
    val count = ObservableField<String>(data)
    val liked = ObservableBoolean()
}

//partial_list_item.xml
<?xml version="1.0" encoding="utf-8"?>
<layout>
    <data>
        <variable
            name="item"
            type="io.ditclear.app.partial.PartialItemViewModel" />

        <variable
            name="presenter"
            type="io.ditclear.bindingadapterx.ItemClickPresenter" />
    </data>

    <androidx.constraintlayout.widget.ConstraintLayout
        android:onClick="@{(v)->presenter.onItemClick(v,item)}"
        >
        <TextView
            android:text="@{item.count}" />
        <ImageView
            android:src="@{item.liked?@drawable/ic_action_liked:@drawable/ic_action_unlike}"/>
    </androidx.constraintlayout.widget.ConstraintLayout>

</layout>
```

当UI需要改变时，改变ItemViewModel的数据即可。

```kotlin
//activity

override fun onItemClick(v: View, item: PartialItemViewModel) {
    item.toggle()
}

// ItemViewModle
 fun toggle(){
        liked.set(!liked.get())

        count.set("$data ${if (liked.get()) "liked" else ""}")
    }
```

至于为什么？这就要说到DataBinding更新UI的原理了。

### DataBinding更新UI原理

当我们在项目中运用DataBinding，并将xml文件转换为DataBinding形式之后。经过编译build，会生成相应的binding文件，比如`partial_list_item.xml`就会生成对于的`PartialListItemBinding`文件，这是一个抽象类，还会有一个`PartialListItemBindingImpl`实现类实现具体的渲染UI的方法`executeBindings`。

当数据有改变的时候，就会重新调用`executeBindings`方法，更新UI，那怎么做到的呢？

![](http://upload-images.jianshu.io/upload_images/3722695-4325e8832fadce97?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

我们先来看`PartialListItemBinding`的构造方法.

```java
public abstract class PartialListItemBinding extends ViewDataBinding {
  @NonNull
  public final ImageView imageView;

  @Bindable
  protected PartialItemViewModel mItem;

  @Bindable
  protected ItemClickPresenter mPresenter;

  protected PartialListItemBinding(DataBindingComponent _bindingComponent, View _root,
      int _localFieldCount, ImageView imageView) {
      //调用父类的构造方法
    super(_bindingComponent, _root, _localFieldCount);
    this.imageView = imageView;
  }
  //... 
 }
```

调用了父类`ViewDataBinding`的构造方法，并传入了三个参数，这里看第三个参数`_localFieldCount`，它代表xml中存在几个`ObservableField`形式的数据，继续追踪.

```java
protected ViewDataBinding(DataBindingComponent bindingComponent, View root, int localFieldCount) {
    this.mBindingComponent = bindingComponent;
    //考点1
    this.mLocalFieldObservers = new ViewDataBinding.WeakListener[localFieldCount];
    this.mRoot = root;
    if (Looper.myLooper() == null) {
        throw new IllegalStateException("DataBinding must be created in view's UI Thread");
    } else {
        if (USE_CHOREOGRAPHER) {
            //考点2
            this.mChoreographer = Choreographer.getInstance();
            this.mFrameCallback = new FrameCallback() {
                public void doFrame(long frameTimeNanos) {
                    ViewDataBinding.this.mRebindRunnable.run();
                }
            };
        } else {
            this.mFrameCallback = null;
            this.mUIThreadHandler = new Handler(Looper.myLooper());
        }

    }
}
```

通过《观察》，发现其根据`localFieldCount`初始化了一个`WeakListener`数组，名为`mLocalFieldObservers`。另一个重点是初始化了一个`mFrameCallback`，在回调中执行了`mRebindRunnable.run`。

当生成的`PartialListItemBindingImpl`对象调用`executeBindings`方法时，通过`updateRegistration`会对`mLocalFieldObservers`数组中的内容进行赋值。

![](http://upload-images.jianshu.io/upload_images/3722695-a536ce90fb676a06?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

随之生成的是相应的`WeakPropertyListener`，来看看它的定义。

```java
private static class WeakPropertyListener extends Observable.OnPropertyChangedCallback
        implements ObservableReference<Observable> {
    final WeakListener<Observable> mListener;

    public WeakPropertyListener(ViewDataBinding binder, int localFieldId) {
        mListener = new WeakListener<Observable>(binder, localFieldId, this);
    }

 	//...
    @Override
    public void onPropertyChanged(Observable sender, int propertyId) {
        ViewDataBinding binder = mListener.getBinder();
        if (binder == null) {
            return;
        }
        Observable obj = mListener.getTarget();
        if (obj != sender) {
            return; // notification from the wrong object?
        }
        //划重点
        binder.handleFieldChange(mListener.mLocalFieldId, sender, propertyId);
    }
}
```

当`ObservableField`的值有改变的时候，`onPropertyChanged`会被调用，然后就会回调`binder`(即binding对象)的`handleFieldChange`方法，继续观察。

```java
private void handleFieldChange(int mLocalFieldId, Object object, int fieldId) {
    if (!this.mInLiveDataRegisterObserver) {
        boolean result = this.onFieldChange(mLocalFieldId, object, fieldId);
        if (result) {
            this.requestRebind();
        }

    }
}
```

如果值有改变 ，result为true，接着`requestRebind`方法被执行。

```java
protected void requestRebind() {
    if (this.mContainingBinding != null) {
        this.mContainingBinding.requestRebind();
    } else {
        synchronized(this) {
            if (this.mPendingRebind) {
                return;
            }

            this.mPendingRebind = true;
        }

        if (this.mLifecycleOwner != null) {
            State state = this.mLifecycleOwner.getLifecycle().getCurrentState();
            if (!state.isAtLeast(State.STARTED)) {
                return;
            }
        }
		//划重点
        if (USE_CHOREOGRAPHER) { // SDK_INT >= 16
            this.mChoreographer.postFrameCallback(this.mFrameCallback);
        } else {
            this.mUIThreadHandler.post(this.mRebindRunnable);
        }
    }

}
```

在上述代码最后，可以看到sdk版本16以上会执行

`this.mChoreographer.postFrameCallback(this.mFrameCallback);`，16以下则是通过`Handler`。

> 关于postFrameCallBack，给的注释是`Posts a frame callback to run on the next frame.`，简单理解就是发生在下一帧即16ms之后的回调。
>
> 关于`Choreographer`，推荐阅读 [Choreographer 解析](https://www.jianshu.com/p/dd32ec35db1d)

但不管如何，最终都是调用`mRebindRunnable.run`，来看看对它的定义。

```java
/**
 * Runnable executed on animation heartbeat to rebind the dirty Views.
 */
private final Runnable mRebindRunnable = new Runnable() {
    @Override
    public void run() {
        synchronized (this) {
            mPendingRebind = false;
        }
        processReferenceQueue();

        if (VERSION.SDK_INT >= VERSION_CODES.KITKAT) {
            // Nested so that we don't get a lint warning in IntelliJ
            if (!mRoot.isAttachedToWindow()) {
                // Don't execute the pending bindings until the View
                // is attached again.
                mRoot.removeOnAttachStateChangeListener(ROOT_REATTACHED_LISTENER);
                mRoot.addOnAttachStateChangeListener(ROOT_REATTACHED_LISTENER);
                return;
            }
        }
        //划重点
        executePendingBindings();
    }
};
```

其实就是在下一帧的时候再执行了一次`executePendingBindings`方法，到这里，DataBinding更新UI的逻辑我们也就全部打通了。

### 写在最后

笔者已经使用了DataBinding好几年的时间，深切的体会到了它对于开发效率的提升，决不下于Kotlin，用好了它就是剑客最锋利的宝剑，削铁如泥，用不好便自损八百。为此，上一年我专门写了一篇[DataBinding实用指南](https://www.jianshu.com/p/015ad08c2c75)讲了讲自己的使用经验，同时开源了运用DataBinding来简化RecyclerView适配器的[BindingListAdapter](https://github.com/ditclear/BindingListAdapter)，希望更多的开发者喜欢它。

GitHub示例：<https://github.com/ditclear/BindingListAdapter>

#### 参考资料

[深入Android databinding的使用和原理分析](https://www.jianshu.com/p/8e393b97f344)

[Choreographer 解析](https://www.jianshu.com/p/dd32ec35db1d)
