---
layout: post
title:  "使用Kotlin构建MVVM应用程序—第三部分：Room"
date:   2019-01-19 10:18:00
categories: Android Kotlin MVVM
---

这是使用Kotlin构建MVVM应用程序—第三部分：Room

在上一篇中我们了解了MVVM是怎么处理网络数据的，而这一篇则介绍的是如何进行数据持久化。

### Room

[Room](https://developer.android.com/topic/libraries/architecture/room.html)是google推出的一个数据持久化库，它是 [Architecture Component](https://developer.android.com/topic/libraries/architecture/index.html)的一部分。它让SQLiteDatabase的使用变得简单，大大减少了重复的代码，并且把SQL查询的检查放在了编译时。

Room使用起来非常简单，而且可以和RxJava配合使用，和我们的技术体系十分契合。

#### 加入依赖

首先在项目的build.gradle中加入

```groovy
allprojects {
    repositories {
        maven {
            url 'https://maven.google.com'
        }
        jcenter()
    }
}
```

接着在app的build.gradle中加入它的依赖

```groovy
//room (local)
implementation 'android.arch.persistence.room:runtime:1.0.0'
implementation 'android.arch.persistence.room:rxjava2:1.0.0'
kapt 'android.arch.persistence.room:compiler:1.0.0'
//facebook出品，可在Chrome中查看数据库
implementation 'com.facebook.stetho:stetho:1.5.0'
```

现在的结构

![MVVM](https://upload-images.jianshu.io/upload_images/3722695-3baa249672d08a92.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/480)



这里我们多了一层Repository，使用这一层来确保单一数据源，保证数据来源的唯一和正确性（即不管是来自网络或是本地缓存的）。ViewModel层并不需要知道它使用到的数据是怎么来的，就好似开发者并不需要知道设计师是如何画出UI图的一样。

开始正文

### 使用Room进行持久化

1.  新建相应的表

Room为每个用[@Entity](https://developer.android.com/reference/android/arch/persistence/room/Entity.html)注解了的类创建一张表

```kotlin
@Entity(tableName = "articles")
class Article(var title: String?){

    @PrimaryKey
    @ColumnInfo(name = "articleid")
    var id: Int = 0
    var content: String? = null
    var readme: String? = null
    @SerializedName("describe")
    var description: String? = null
    var click: Int = 0
    var channel: Int = 0
    var comments: Int = 0
    var stow: Int = 0
    var upvote: Int = 0
    var downvote: Int = 0
    var url: String? = null
    var pubDate: String? = null
    var thumbnail: String? = null
}
```

2 .  创建相关的Dao

*相当于Retrofit中的api接口*

[DAO](https://developer.android.com/topic/libraries/architecture/room.html#daos)负责定义操作数据库的方法。在SQLite实现的版本中，所有的查询都是在LocalUserDataSource文件中完成的，里面主要是 使用了Cursor对象来完成查询的工作。有了Room，我们不再需要Cursor的相关代码，而只需在Dao类中使用注解来定义查询。

```kotlin
@Dao
interface PaoDao{

    @Insert(onConflict = OnConflictStrategy.REPLACE)
    fun insetAll(articles: List<Article>)

    @Query("SELECT * FROM Articles WHERE articleid= :id")
    fun getArticleById(id:Int):Single<Article>


    @Insert(onConflict = OnConflictStrategy.REPLACE)
    fun insertArticle(article :Article)

}
```

3 . 创建数据库

   *相当于创建RetrofitClient对象*

   我们需要定义一个继承了RoomDatabase的抽象类。这个类使用@Database来注解，列出它所包含的Entity以及操作它们的 DAO 。

   ```kotlin
   @Database(entities = arrayOf(Article::class),version = 1)
   abstract class AppDatabase :RoomDatabase(){

       abstract fun paoDao(): PaoDao

       companion object {
           @Volatile private var INSTANCE: AppDatabase? = null
           fun getInstance(context: Context): AppDatabase =
                   INSTANCE ?: synchronized(this) {
                       INSTANCE ?: buildDatabase(context).also { INSTANCE = it }
                   }

           private fun buildDatabase(context: Context) =
                   Room.databaseBuilder(context.applicationContext,
                           AppDatabase::class.java, "app.db")
                           .build()
       }

   }
   ```

Over ，集成Room十分的简单。

更多关于Room的使用方法，它的迁移，表之间的关联和字段。

推荐查看泡网的Room专题：[Room](http://www.jcodecraeer.com/a/anzhuokaifa/androidkaifa/2017/0728/8279.html)

### 实践

这里我们对上一篇中的从服务器端获取到的Article文章进行持久化。

1. 修改一下Model层的代码，添加Repository作为ViewModel层的数据源

```kotlin
class PaoRepo constructor(private val remote:PaoService,private val local :PaoDao){
	//首先查看本地数据库是否存在该篇文章
    fun getArticleDetail(id:Int)= local.getArticleById(id)
            .onErrorResumeNext {
              	//本地数据库不存在，会抛出EmptyResultSetException
              	//转而获取网络数据,成功后保存到数据库
                remote.getArticleDetail(id)
                        .doOnSuccess { local.insertArticle(it) }
            }
}
```

我们的目录结构会如下图所示：

![结构](https://upload-images.jianshu.io/upload_images/3722695-6713c9428d0586e0.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/360)

2 . 修改我们的ViewModel层的数据源

在上一篇中我们使用的是`PaoService`网络数据作为数据源，这里只需要修改为`PaoRepo`

```kotlin
class PaoViewModel(private val repo: PaoRepo) 
```

以后统一使用PaoRepo来为PaoViewModel提供数据

3 . 在View层中将`PaoRepo`注入到`PaoViewModel`

```kotlin
//////model
val remote=Retrofit.Builder()
        .baseUrl(Constants.HOST_API)
        .addCallAdapterFactory(RxJava2CallAdapterFactory.create())
        .addConverterFactory(GsonConverterFactory.create())
        .build().create(PaoService::class.java)
val local=AppDatabase.getInstance(applicationContext).paoDao()
val repo = PaoRepo(remote, local)
/////ViewModel
mViewMode= PaoViewModel(repo)
////binding
mBinding.vm=mViewMode
```



看看效果

![image](http://upload-images.jianshu.io/upload_images/3722695-da10c9f98784fde4?imageMogr2/auto-orient/strip)

### Roomigrant

> Roomigrant is a helper library to automatically generate Android Room library migrations using compile-time code generation.

这是一个帮助开发者进行Room数据迁移的库，使用起来非常方便。

在使用Room的过程中，或多或少都会遇到增加表、改变表字段的情况。但是Room的迁移比较麻烦，可以看看[理解Room的数据迁移](http://www.jcodecraeer.com/a/anzhuokaifa/androidkaifa/2017/0728/8278.html)这篇文章，增加新表还好，只需要修改版本号，但是如果需要修改表字段的话，由于sqlite的历史原因：SQLite的ALTER TABLE命令[非常局限](https://sqlite.org/lang_altertable.html)，只支持重命名表以及添加新的字段。

因此我们需要：

- 创建一个新的临时表，
- 把users表中的数据拷贝到临时表中
- 丢弃users表
- 把临时表重命名为users

使用Room，Migration的实现是这样的：

```kotlin
static final Migration MIGRATION_3_4 = new Migration(3, 4) {
    @Override
    public void migrate(SupportSQLiteDatabase database) {
        // Create the new table
        database.execSQL(
                "CREATE TABLE users_new (userid TEXT, username TEXT, last_update INTEGER, PRIMARY KEY(userid))");
// Copy the data
        database.execSQL(
                "INSERT INTO users_new (userid, username, last_update) SELECT userid, username, last_update FROM users");
// Remove the old table
        database.execSQL("DROP TABLE users");
// Change the table name to the correct one
        database.execSQL("ALTER TABLE users_new RENAME TO users");
    }
};
```

对于开发者还需要编写大量的sql语句，可以说是不友好了。

但是幸运的是[Roomigrant](https://github.com/MatrixDev/Roomigrant)帮助我们做到了以上的事，对于修改表字段，只需要简单的

自定义一个Rule，通知一下什么表下的哪个字段需要修改

```kotlin
// version 3 的users表的字段uId
// version 4 相应的字段改为了userId
@FieldMigrationRule(version1 = 3, version2 = 4, table = "users", field = "uId")
fun migrate_3_4_Object1Dbo_intVal(): String {
	return "`users`.`userId`"
}
```

这样[Roomigrant](https://github.com/MatrixDev/Roomigrant)便可以帮助我们生成那些模板式的代码。

### 写在最后

本项目的github地址：[https://github.com/ditclear/MVVM-Android](https://link.jianshu.com/?t=https://github.com/ditclear/MVVM-Android)

更多的例子可以查看：[https://github.com/ditclear/PaoNet](https://link.jianshu.com/?t=https://github.com/ditclear/PaoNet)

这是使用Kotlin构建MVVM项目的第三部分，主要讲了怎么在MVVM中进行数据的持久化以及为ViewModel层提供Repository作为唯一的数据源。

总结一下前三篇的内容便是:

> 使用Retrofit提供来自服务端的数据，使用Room来进行持久化，然后提供一个Repository来为ViewModel提供数据，ViewModel层利用RxJava来进行数据的转换，配合DataBinding引起View层的变化。

逻辑很清晰了，逻辑很清晰了，下一篇，我们将会了解到google出品的jetpack工具ViewModel组件，敬请期待。