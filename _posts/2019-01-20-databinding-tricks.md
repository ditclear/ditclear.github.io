---
layout: post
title:  "DataBinding实用指南"
date:   2019-01-20 10:18:00
categories: Android databinding
---

对于android开发者而言，写冗余重复的代码一直是一件吃力不讨好的事情，而数据绑定技术能够减少大量重复的代码，可以说是android开发者的福音。它学习起来十分简单（相信了解过的应该都这么觉得），但使用起来却不那么尽如人意（对不起，binding文件未找到）。

从16年11月到现在，经过这么长时间的实践，除了前4个月在踩坑之外，到现在都没再遇到DataBinding相关的错误，趁年前有些时间，因此总结了一下实际项目中使用DataBinding的一些经验。

如果你对此稍有兴趣，可以看看我以前的文章[告别findView和ButterKnife](https://www.jianshu.com/p/499c8e2b80c4)

如果你想学习DataBinding，推荐看看[泡网的DataBinding专题](http://www.jcodecraeer.com/a/anzhuokaifa/androidkaifa/2015/0811/3290.html)或者慕课网的视频

如果你正在使用DataBinding并觉得苦恼，那么看本文是一个不错的选择。

##### 相关代码：

完整示例：https://github.com/ditclear/PaoNet

DataBinding-AspectJ:https://github.com/ditclear/DataBinding-AspectJ

### 正文

- #### 首先推荐一款AS插件[DataBinding Support](https://plugins.jetbrains.com/plugin/9271-databinding-support)

  可以简化DataBinding的转换操作并支持和`ViewModel`和与之关联的layout文件的跳转，可以提升开发时的效率，节省时间

  贴两张图看看：

  ![](https://raw.githubusercontent.com/shiraji/databinding-support/master/websites/images/wrap.gif)

  ![](https://raw.githubusercontent.com/shiraji/databinding-support/master/websites/images/jump_to_layout.gif)

更多功能可查看此链接：https://plugins.jetbrains.com/plugin/9271-databinding-support

- #### 统一命名的variable

  俗话说无法不成章，对于一个团队而言，不管是大还是小，都需要一套合理的统一的命名规范，既方便相互之间的协作，也减轻了CodeReview时的困难。对实践了Databinding的团队格外如此。

  ##### bad:

  ```xml
  <!--user_activity.xml -->
  <variable
                  name="uservm"
                  type="io.ditclear.app.viewmodel.UserViewModel"/>
  <!--student_activity.xml -->
  <variable
                  name="studentvm"
                  type="io.ditclear.app.viewmodel.StudentViewModel"/>
  
  ```

  ##### better:

  ```xml
  <!--user_activity.xml -->
  <variable
                  name="vm"
                  type="io.ditclear.app.viewmodel.UserViewModel"/>
  <!--student_activity.xml -->
  <variable
                  name="vm"
                  type="io.ditclear.app.viewmodel.StudentViewModel"/>
  ```

- #### 尽可能少的variable和import

  也许你是刚接触DataBinding，按照官网的Guide，很可能会定义一些非必需的variable或者import 一些自定义的静态类来进行字符串处理、View显示隐藏等等的操作。但是这不是一个好习惯，过多的variable除了会让你多做几次无谓的绑定外，数据也将变得难以管理。所以尽可能少的variable和import是一种较好的实践。

  ##### bad:

  ```xml
  <data>
  
          <import type="com.ditclear.app.util.DateUtil"/>
  
          <import type="android.view.View"/>
  
          <import type="com.ditclear.app.util.StringUtil"/>
  
          <import type="com.ditclear.app.network.model.StudentState"/>
  
          <import type="java.math.BigDecimal"/>
  
          <import type="com.ditclear.app.presentation.student.StudentActivity"/>
  
          <variable
              name="isShow"
              type="Boolean"/>
  
          <variable
              name="time"
              type="java.util.Date"/>
  
          <variable
              name="date"
              type="java.util.Date"/>
  
          <variable
              name="signTime"
              type="String"/>
  
          <variable
              name="item"
              type="com.ditclear.app.network.model.student.StudentItem"/>
  
          <variable
              name="presenter"
              type="com.ditclear.app.presentation.student.StudentActivity.Presenter"/>
      </data>
  ```

  ##### better:

  ```xml
  <data>
          <variable
                  name="presenter"
                  type="com.ditclear.app.helper.presenter.Presenter"/>
  
          <variable
                  name="vm"
                  type="com.ditclear.app.view.student.viewmodel.StudentViewModel"/>
      </data>
  ```

  数据的处理和view的显示都用ViewModel来处理，比如：

  ```xml
  android:text="@{vm.signTime}"
  android:visibility="@{vm.isShowVisbility}"
  ```

  这样的好处是减轻了layout.xml文件的复杂程度，xml文件只用来单纯的展示数据， 而复杂的逻辑判断都放在了ViewModel中，并且为页面布局数据提供了唯一的入口，方便查看和管理数据源。

- #### 避免使用复杂的表达式

  ##### bad:

  在官方的指南里有这样的写法：

  ```xml
  <data>
      <import type="com.example.MyStringUtils"/>
      <variable name="user" type="com.example.User"/>
  </data>
  
  <TextView
     android:text="@{MyStringUtils.capitalize(user.lastName)}"
     android:layout_width="wrap_content"
     android:layout_height="wrap_content"/>
  ```

  或者这样的

  ```xml
  android:text="@{String.valueOf(index + 1)}"
  android:visibility="@{age < 13 ? View.GONE : View.VISIBLE}"
  android:transitionName='@{"image_" + id}'
  android:text="@{@string/nameFormat(firstName, lastName)}"
  android:padding="@{large? @dimen/largePadding : @dimen/smallPadding}"
  android:text="@{map[`firstName`]}"
  ```

  **需要注意的是这些都不代表着最佳实践，只是说明有这样的功能，支持这样的用法。**

  **而且它有一个很大的弊端，相信很多刚入门DataBinding的人都遇到过找不到binding文件的错，然后被劝退，原因无外乎就是表达式写错、variable的类路径不对或者没import相应的类（比如View）等等，都与复杂的表达式有关系，而DataBinding报错也不太智能，有时不能准确定位，所以建议不要在xml里进行复杂的数据绑定，这些都尽量放到ViewModel里进行，xml只绑定简单的基础数据类型。**

  ##### better：

  ```xml
  <data>
      <variable name="vm" type="com.example.UserViewModel"/>
  </data>
  
  <TextView
     android:text="@{vm.capitalizeLastName}"
     android:layout_width="wrap_content"
     android:layout_height="wrap_content"
     android:hint="@{vm.index}"
     android:visibility="@{vm.showName}"
     android:transitionName='@{vm.transitionName}'/>
  ```

- #### 点击事件的命名和处理

  先看看DataBinding支持的写法

  ```java
  android:onClick="@{presenter.onClick()}" //1.方法引用
  android:onClick="@{()->presenter.onClick()}" //2.lamda表达式
  android:onClick="@{(view)->presenter.onClick(view)}" //3.lamda表达式
  android:onClick="@{()->presenter.onClick(item)}"//4.带参数lamda表达式
  android:onClick="@{(view)->presenter.onClick(view, item)}"//5.带参数lamda表达式
  ```

  选择很多，而且使用方法也是五花八门，有的喜欢直接使用viewmodel里的方法，有的直接将Activity或者fragment作为handler，还有的可能会在activity/fragment里写一个内部类作为presenter，然后由于方法名也可以自定义，所以很可能出现你叫`presenter.save()`另一个叫`(v)->handler.onSave(v)`的情况。

  ##### bad：

  ```xml
  <layout>
  <data>
  
          <variable
                  name="vm"
                  type="io.ditclear.app.viewmodel.AnimalViewModel"/>
          
          <variable
                  name="handler"
                  type="io.ditclear.app.view.AnimalActivity"/>
  
          <variable
                  name="presenter"
                  type="io.ditclear.app.view.AnimalActivity.Presenter"
  
      </data>
  
      <LinearLayout
              tools:context="io.ditclear.app.view.AnimalActivity">
          <Button
                  android:onClick="@{vm.shoutWhat()}"/>
  
          <Button
                  android:onClick="@{(v)->handler.shout(v)}"/>
          <Button
                  android:onClick="@{()->presenter.onShout()}"/>
    </LinearLayout>
    </layout>
  ```

  ##### better：

  推荐的一种处理方式是使用封装过的`View.OnClickListener`来统一处理点击事件，包裹一层的目的是为了不依赖于具体实现。

  ```java
  public interface Presenter extends View.OnClickListener{   
      @Override
      void onClick(View v);
  }
  ```

  xml布局文件

  ```xml
  <layout>
  <data>
  
          <variable
                  name="vm"
                  type="io.ditclear.app.viewmodel.AnimalViewModel"/>
          <variable
                  name="presenter"
                  type="io.ditclear.app.helper.Presenter"
  
      </data>
  
      <LinearLayout
              tools:context="io.ditclear.app.view.AnimalActivity">
          <Button
                  android:id="@+id/save_btn"   
                  android:onClick="@{(v)->presenter.onClick(v)}"/>
  
          <Button
                  android:id="@+id/delete_btn"   
                  android:onClick="@{(v)->presenter.onClick(v)}"/>
          <Button
                  android:id="@+id/submit_btn"   
                  android:onClick="@{(v)->presenter.onClick(v)}"/>
    </LinearLayout>
    </layout>
  ```

  **这里推荐使用(v)->presenter.onClick(v)的写法，原因之一是比较直观一点，其二是需要参数view**

  接着在activity/fragment中来实现Presenter接口，处理点击事件

  ```java
  public class AnimalActivity extends AppCompatActivity implements Presenter {
  
      private AnimalActivityBinding mBinding;
  	private AnimalViewModel mViewModel;
      @Override
      protected void onCreate(@Nullable Bundle savedInstanceState) {
          super.onCreate(savedInstanceState);
          mBinding= DataBindingUtil.setContentView(this, R.layout.animal_activity);
          Animal animal=new Animal("dog",1);
          mViewModel=new AnimalViewModel(animal);
          mBinding.setVm(viewModel);
          mBinding.setPresenter(this);
      }
  
    	@SingleClick
      @Override
      public void onClick(View v) {
        	//根据id进行区分
          switch (v.getId()){
              case R.id.save_btn:
                  save();
                  break;
              case R.id.delete_btn:
                  delete();
                  break;
              case R.id.submit_btn:
                  submit();
                  break;
          }
      }
    
    	private void submit(){
          //调用viewModel的方法
        	mViewModel.submit();
      }
  }
  ```

  @SingleClick是一个注解，作为AspectJ的切面，来防止多次点击，需要将view作为参数，详细可参考文章[DataBinding结合AspectJ防止多次点击](https://www.jianshu.com/p/ea45670a364f)

  如果你使用RxJava和RxLifeCycle来处理数据和管理生命周期，那么这里的`submit()`方法将会更加简单。

  ```java
  private void submit(){
          //调用viewModel的方法
        	mViewModel.submit()
            .compose(bindToLifecycle())
            .subscribe({
                //success
            },{
                //error
            })
      }
  ```

  详细情况可以看这篇文章：[Retrofit及RxJava](https://www.jianshu.com/p/8993b247947a)

- #### 处理RecyclerView 列表项的数据及点击事件

  RecyclerView功能极其强大，能做到的事情很多，网上已经出现很多关于多类型RecyclerView的处理方法，在使用DataBinding这一年多时间里，感受便是使用DataBinding来处理RecyclerView Item再合适不过，充分做到了数据和itemView的完美分离，告别了反复、冗余的自定义Adapter，不需要关心太多无意义的事情。

  详情请查看github地址（kotlin版本）：https://github.com/ditclear/BindingListAdapter

### 一些技巧

- #### 使用tools来进行预览

  tools可以告诉Android Studio，哪些属性在运行的时候是被忽略的，只在设计布局的时候有效。比如我们要让android:text属性只在布局预览中有效可以这样

  ```xml
  <TextView
   android:id="@+id/text_main"
   android:layout_width="match_parent"
   android:layout_height="wrap_content"
   android:textAppearance="@style/TextAppearance.Title"
   android:layout_margin="@dimen/main_margin"
   android:text="@{vm.title}"
   tools:text="I am a title" />
  ```

  tools可以覆盖android的所有标准属性，将android:换成tools:即可。同时在运行的时候就连tools:本身都是被忽略的，不会被带进apk中。

- #### 补全自定义的属性

  比如为ImageView，你定义了一个BindingAdapter

  ```java
  	@BindingAdapter("url")
      public static void bindImgUrl(ImageView imageView,String url){
          Glide.with(imageView.getContext()).load(url).into(imageView);
      }
  ```

  实际情况中，ImageView并没有url这个属性， 这时可以在attrs.xml文件中为ImageView添加这一属性，rebuild一下项目，以后就能自动补全属性了

  ```xml
  <declare-styleable name="ImageView">
          <attr name="url" format="string"/>
      </declare-styleable>
  ```

  ![attr](http://upload-images.jianshu.io/upload_images/3722695-bf3c97bb1dde3da3.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)



### 最后

DataBinding使用起来很简单，但是由于它没有一个统一的规范和写法，需要靠开发者自己去摸索和研究才能熟练运用，而这其中又会出现一些小坑，所以可能导致刚学习的人觉得难以驾驭而被迫放弃。但是它却是一门非常实用的技术，不管你是否使用MVVM架构，单凭它可以减少很多冗余的代码和跟RecyclerView的完美契合的优点就值得去了解和使用它。

关于怎么较好的实践，总结一哈：

- 统一命名规范
- xml文件中避免复杂的表达式
- xml只负责展示文本数据，数据的处理和view的显示隐藏交给ViewModel去做
- 点击事件封装一哈，在Activity/Fragment中去处理事件或调用ViewModel的方法

在经过一年以上实践后，总结出了以上的一些避免踩坑的方式和较好的实践方法，希望对准备学习、正在学习或者正在使用的同学一些帮助。

毕竟对于DataBinding  ：使用得当，那它就是神兵利器，使用不当，那么便伤人(Code)伤己。









