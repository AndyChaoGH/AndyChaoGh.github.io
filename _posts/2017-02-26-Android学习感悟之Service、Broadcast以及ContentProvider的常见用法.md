---
layout: post
title: Android感悟之Service、Broadcast以及ContentProvider的常见用法
author: arvinljw
---

本篇包含Service、Broadcast以及ContentProvider的常见用法

# 简介

四大组件，包含了Activity、Service、Broadcast以及ContentProvider，而Activity使用最多，所以在感悟的第一篇就介绍了；而当我们在项目开发过程中，就难免会用到其他的组件，这篇文章中就会讲道如何使用其他组件，并且有一些个人的理解。

## Service

### Service生命周期

Service，我们又叫它服务，这个东西在进程间通信的文章中，就帮了我们大忙，通过绑定服务，得到Binder对象，就能够进程间的通信了，而这只是启动服务的一种方式，还有一种就是通过context来startService。

服务和Activity一样，都是有生命周期的，下面就上一张官方Service的生命周期图。

![](../images/service_lifecycle.png)

可以看到Service的生命周期都是以onCreate开始，onDestory结束，但是使用不同的启动方式去启动服务，生命周期的中间是会有不同的地方。

总的来看Service中的回调方法有如下几个：

1、**onCreate**：只有在第一次创建（不管如何创建）的时候，才会被回调

2、**onStartCommand**：只有通过startService启动的服务才回回调；

3、**onBind**：只有第一次通过bindService启动的服务才回回调，之后再绑定也不回调了，因为该方法其实就是去返回一个已经创建好的Binder；

4、**onUnbind**：只有最后一个unbindService的时候调用

5、**onDestory**：只有在所有启动都结束后才会回调,即startService多少次就要stopService多少次，才能被结束；或者bindService多少次就要unBindService多少次才能被结束；或者两者都有，即启动和结束要匹配完才能结束。

**注：具体的测试demo的地址会在最后放上**

### 前台Service

大家都知道Service是没有界面的，给人的感觉就是后台运行的，正常情况的确如此，但是它却依旧运行在主线程，所以耗时操作都要在子线程中处理，其实Service还有一种叫做前台服务，就好比网易云音乐的播放操作栏。

其必要条件是：必须在状态栏提供通知，除非服务停止或从前台移除，否则不能清除该通知。

而具体的方法就是调用startForeground()方法，其两个参数为唯一标识通知的整型数(**不能为0**)和状态栏的Notification，例如：

```
NotificationCompat.Builder builder = new NotificationCompat.Builder(this);
builder.setSmallIcon(R.mipmap.ic_launcher);
builder.setLargeIcon(BitmapFactory.decodeResource(getResources(), R.mipmap.ic_launcher));
builder.setAutoCancel(false);
builder.setOngoing(true);
builder.setShowWhen(true);
builder.setContentTitle("这是一个前台服务");
builder.setContentText("你好啊～");
Notification notification = builder.build();
startForeground(NOTIFICATION_DOWNLOAD_PROGRESS_ID, notification);
```

而如果该通知要做的更好，就需要用到RemoteView的知识，可能后续的文章中会讲道。

在前台demo中既然能start就肯定会有stopForeground(true)，参数表示，是否移除该通知，而该通知的开和关，这里让启动服务的组件去控制，便用到了Messenger，这是一种比较简单的进程间通信的方法，具体的代码就不上了，想了解的就看看demo去吧，代码比较简单。

## Broadcast

广播，在目前的开发中用到的不多，用到的地方就如：软件的安装情况、极光推送的回调、短信监听等等；

广播的回调方法只有一个onReceive(Context context, Intent intent)，且在该方法中，操作不能超过10秒，否则会ANR。参数的具体含义就不多解释了。

广播有两种注册方式，一种是静态注册，一种是动态注册，下面分辨来看看如何实现的：

1、静态注册：

需要我们在AndroidManifest.xml文件中，注册receiver，所以我们就得先实现一个：

```
import android.content.BroadcastReceiver;
import android.content.Context;
import android.content.Intent;
import android.widget.Toast;

/**
 * created by arvin on 17/2/24 00:02
 * email：1035407623@qq.com
 */
public class StaticReceiver extends BroadcastReceiver {
    @Override
    public void onReceive(Context context, Intent intent) {
        Toast.makeText(context, "静态注册的广播", Toast.LENGTH_SHORT).show();
    }
}
```

然后在AndroidManifest.xml中声明：

```
<receiver android:name=".broadcast.StaticReceiver">
    <intent-filter>
        <action android:name="net.arvin.androidart.broadcast.static" />
    </intent-filter>
</receiver>
```

这样就注册成功了，这里有个细节，exported没有设置时，如果有intent-filter则表示，该广播是可以被其他进程访问的，反之则不能被其他进程方法；如果有exported则以其值为准；

2、动态注册：

这种方式其实也是一样只是声明方式不同，首先依然时实现一个广播接收者：

```
private BroadcastReceiver mReceiver = new BroadcastReceiver() {
    @Override
    public void onReceive(Context context, Intent intent) {
        showToast("动态注册的广播");
    }
};
```

然后通过context的registerReceiver方法注册：

```
registerReceiver(mReceiver, getFilter());

private IntentFilter getFilter() {
    IntentFilter filter = new IntentFilter();
    filter.addAction(DYNAMICS_ACTION);
    return filter;
}
```

动态注册的广播，需要在不用的时候调用unregisterReceiver，例如在onDestroy方法中调用，避免内存泄露。

这样两种注册方式简单的介绍了一下。

然后发送广播，其实这个也很简单，发送的广播有有序和无序两种，首先来看无需的。

1、发送无序广播：

只需调用context的sendBroadcast即可，传入的事intent，这里就涉及到intent-filter的匹配规则，匹配到则发送到。具体如何匹配可以查看[2017-02-03-Android感悟之Intent.md](2017-02-03-Android感悟之Intent.md)文章中的讲到的匹配规则。

2、发送有序广播：

只需要调用context的sendOrderedBroadcast即可，而所谓有序广播则是依次调用注册的广播，排序规则就是在广播注册的时候，设置的filter.setPriority(int priority);参数的范围事[-1000,1000]其中值越大，就越先收到广播，然后先收到的还能改变intent的值，或者是拦截掉该广播，例如拦截在黑名单中的电话的短信或来电就是这个原理。拦截的方法也很简单就是在准备拦截的广播中调用abortBroadcast()方法即可。

## ContentProvider用法

ContentProvider它是一种系统提供的重要的进程间数据交互的方式，底层的实现也是Binder，具体的实现这里也就不介绍了，先看看怎么用。交互肯定分为至少两方，一方是数据提供者（实现ContentProvider），一方则是数据访问者（ContentResolver），而这两者的交互过程中又有一个最为重要的东西Uri以及UriMatcher，统一资源标识符，下面也会挨着介绍。。

### Uri

这是一种用于标识某一互联网资源名称的字符串，它有三部分，在ContentProvider中大概可以这么分：content://<authority>/<path>

* "content://"：这是第一部分，是协议;
* \<authority>：这是第二部分，是主机IP；
* \<path>：这是第三部分，是路径，方便我们用来判断怎么操作的部分。

### 数据访问者

系统提供了ContentResolver类方便用户通过Uri访问ContentProvider中的提供的内容，而数据的操作无外乎增删查改，刚好这个类也提供了这几种方法，下面就先简单的介绍一下，具体如何用请参考demo：

1、query：查，其中参数有5个比较重要下面挨着说：

* Uri uri：这个是访问数据的路径；
* String[] projection：这个是查询需要返回的对应列的数据，null则表示所有列；
* selection：这个是表示sql语句where后边的条件；
* String[] selectionArgs：这个是条件中占位符对应的参数值；
* sortOrder：这个是sql语句中order by后边的排序方式。

2、insert：增，这个是插入单条数据，参数：

* Uri uri：这个是访问数据的路径；
* ContentValues values：以map的方式用来存储插入的数据，key是列名，value是值。

当然也可以插入多条调用bulkInsert，只需要把第二个参数变成数组即可；

3、update：改，修改数据，参数：

* Uri uri：这个是访问数据的路径；
* ContentValues values：以map的方式用来存储插入的数据，key是列名，value是值。
* String where：这个是表示sql语句where后边的条件，用于筛选；
* String[] selectionArgs：这个是条件中占位符对应的参数值；

4、delete：删，删除数据，参数：

* Uri uri：这个是访问数据的路径；
* String where：这个是表示sql语句where后边的条件，用于筛选；
* String[] selectionArgs：这个是条件中占位符对应的参数值；

到这里增删查改四个方法的参数也介绍的差不多了，具体如何使用请参考demo中的ProviderActivity和AddUserActivity。

### ContentProvider

使用ContentProvider，则有两部，第一继承ContentProvider，第二在清单文件中配置provider属性，记得设置authorities属性，第二部没什么好说的。来看第一部：

继承ContentProvider，需要重写6个方法：

* onCreate：在第一次被调用时创建，一般用于初始化工作
* getType：这个是用于匹配Intent中的mimeType属性的，当然必须要让ContentProvider在清单文件的中的过滤器中设置该属性才有用；
* query：查询；
* insert：添加；
* update：修改；
* delete：删除。

而增删查改所操作的数据可以是，可以是数据库、文件系统或网络；而更多使用到的是数据库，所以这篇就以数据库为例；下面就以User表为例，包含三个字断：id，name，age；id自增；创建表，我们使用现在比较流行的Greendao，3.0以后使用起来更加方便，使用注解即可，它和ButterKnife的原理一样，有预编译，也不会影响效率，方法如下：

1、在项目级的build.gradle文件中加入：

```
dependencies {
	classpath 'org.greenrobot:greendao-gradle-plugin:3.2.1'
}
```

2、在module级的build.gradle文件中加入：

```
apply plugin: 'org.greenrobot.greendao'

greendao {//这是表示生成的包的目录以及数据库的版本
    schemaVersion 1
    daoPackage 'net.arvin.androidart.gen'
    targetGenDir 'src/main/java'
}

dependencies {
    //数据库
    compile 'org.greenrobot:greendao:3.2.0'
    compile 'org.greenrobot:greendao-generator:3.2.0'
}
```

等下载完就配置完成，然后就能直接写实体：

```
@Entity
public class User implements Parcelable {
    @Id(autoincrement = true)
    private Long id;
    private String name;
    private int age;

    @Generated(hash = 1309193360)
    public User(Long id, String name, int age) {
        this.id = id;
        this.name = name;
        this.age = age;
    }

    @Generated(hash = 586692638)
    public User() {
    }

    public Long getId() {
        return id;
    }

    public void setId(Long id) {
        this.id = id;
    }

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

    public int getAge() {
        return age;
    }

    public void setAge(int age) {
        this.age = age;
    }

    @Override
    public int describeContents() {
        return 0;
    }

    @Override
    public void writeToParcel(Parcel dest, int flags) {
        dest.writeValue(this.id);
        dest.writeString(this.name);
        dest.writeInt(this.age);
    }

    protected User(Parcel in) {
        this.id = (Long) in.readValue(Long.class.getClassLoader());
        this.name = in.readString();
        this.age = in.readInt();
    }

    public static final Parcelable.Creator<User> CREATOR = new Parcelable.Creator<User>() {
        @Override
        public User createFromParcel(Parcel source) {
            return new User(source);
        }

        @Override
        public User[] newArray(int size) {
            return new User[size];
        }
    };
}
```

写好实体后，rebuild一下就生成好了，在net.arvin.androidart.gen包下会多出
DaoMaster、DaoSession、UserDao三个类；

这个实体其中构造方法是greendao生成的，然后序列化是因为，在demo中需要传递数据；

他的那些注解这里就不介绍了，有兴趣的可以自行Google。

然后我们来看ContentProvider的实现：

这时候我们需要解决一个问题，数据访问者发来的Uri，正确性我们应该如何判断呢？这里便引入了UriMatcher这个工具类，先来看使用方式：

```
public static final String MIME_ITEM = "user";

public static final String AUTHORITY = "net.arvin.androidart";
public static final String PATH_SINGLE = MIME_ITEM + "/#";
public static final String PATH_MULTIPLE = MIME_ITEM;

private static final int MULTIPLE_PEOPLE = 1;
private static final int SINGLE_PEOPLE = 2;
private static final UriMatcher uriMatcher;

static {
    uriMatcher = new UriMatcher(UriMatcher.NO_MATCH);
    uriMatcher.addURI(AUTHORITY, PATH_MULTIPLE, MULTIPLE_PEOPLE);
    uriMatcher.addURI(AUTHORITY, PATH_SINGLE, SINGLE_PEOPLE);
}
```

在访问者传入Uri是，即可通过uriMatcher.match(uri)，返回code，表示对应的哪一个操作，这里有两种操作操作单条，操作全部；其中uriMatcher.addUri的参数，分别是：

* String authority：这个是Uri中的协议；
* String path：这个是Uri中的路径，其中#可表示任意数字，而这里这个数字就是下面会用到的id。
* int code：这个是Uri中路径对应的code，匹配上就返回这个值。

下面就来看看全部的实现源码：

```
import android.content.ContentProvider;
import android.content.ContentUris;
import android.content.ContentValues;
import android.content.UriMatcher;
import android.database.Cursor;
import android.database.SQLException;
import android.database.sqlite.SQLiteDatabase;
import android.database.sqlite.SQLiteQueryBuilder;
import android.net.Uri;
import android.support.annotation.NonNull;

import net.arvin.androidart.gen.DaoMaster;
import net.arvin.androidart.gen.UserDao;

/**
 * created by arvin on 17/2/25 15:10
 * email：1035407623@qq.com
 */
public class UserProvider extends ContentProvider {
    //mimeType
    public static final String MIME_DIR_PREFIX = "vnd.android.cursor.dir";
    public static final String MIME_ITEM_PREFIX = "vnd.android.cursor.item";
    public static final String MIME_ITEM = "user";

    public static final String MIME_TYPE_SINGLE = MIME_ITEM_PREFIX + "/" + MIME_ITEM;
    public static final String MIME_TYPE_MULTIPLE = MIME_DIR_PREFIX + "/" + MIME_ITEM;

    //有效Uri
    public static final String AUTHORITY = "net.arvin.androidart";
    public static final String PATH_SINGLE = MIME_ITEM + "/#";
    public static final String PATH_MULTIPLE = MIME_ITEM;

    private static final int MULTIPLE_PEOPLE = 1;
    private static final int SINGLE_PEOPLE = 2;
    private static final UriMatcher uriMatcher;

    static {
        uriMatcher = new UriMatcher(UriMatcher.NO_MATCH);
        uriMatcher.addURI(AUTHORITY, PATH_MULTIPLE, MULTIPLE_PEOPLE);
        uriMatcher.addURI(AUTHORITY, PATH_SINGLE, SINGLE_PEOPLE);
    }

    public static final String CONTENT_URI_STRING = "content://" + AUTHORITY + "/" + PATH_MULTIPLE;
    public static final Uri CONTENT_URI = Uri.parse(CONTENT_URI_STRING);

    private SQLiteDatabase db;

    @Override
    public boolean onCreate() {
        DaoMaster.DevOpenHelper mHelper = new DaoMaster.DevOpenHelper(getContext(), "art-db", null);
        db = mHelper.getWritableDatabase();
        return db != null;
    }

    @Override
    public String getType(@NonNull Uri uri) {
        switch (uriMatcher.match(uri)) {
            case MULTIPLE_PEOPLE:
                return MIME_TYPE_MULTIPLE;
            case SINGLE_PEOPLE:
                return MIME_TYPE_SINGLE;
            default:
                throw new IllegalArgumentException("UnKnow uri:" + uri);
        }
    }

    @Override
    public Uri insert(@NonNull Uri uri, ContentValues values) {
        long id = db.insert(UserDao.TABLENAME, null, values);
        if (id > 0) {
            Uri newUri = ContentUris.withAppendedId(CONTENT_URI, id);
            notifyChange(newUri);
            return newUri;
        }
        throw new SQLException("failed to insert row into " + uri);
    }

    @Override
    public Cursor query(@NonNull Uri uri, String[] projection, String selection, String[] selectionArgs, String sortOrder) {
        SQLiteQueryBuilder qb = new SQLiteQueryBuilder();

        qb.setTables(UserDao.TABLENAME);
        qb.appendWhere(getWhere(uri, selection));

        Cursor cursor = qb.query(db, projection, selection, selectionArgs, null, null, sortOrder);

        notifyChange(uri);
        return cursor;
    }

    @Override
    public int delete(@NonNull Uri uri, String selection, String[] selectionArgs) {
        int count = db.delete(UserDao.TABLENAME, getWhere(uri, selection), selectionArgs);

        notifyChange(uri);
        return count;
    }

    @Override
    public int update(@NonNull Uri uri, ContentValues values, String selection, String[] selectionArgs) {
        int count = db.update(UserDao.TABLENAME, values, getWhere(uri, selection), selectionArgs);
        notifyChange(uri);
        return count;
    }

    private String getWhere(@NonNull Uri uri, String selection) {
        String where;
        switch (uriMatcher.match(uri)) {
            case SINGLE_PEOPLE:
                where = UserDao.Properties.Id.columnName + "=" + uri.getPathSegments().get(1);
                break;
            case MULTIPLE_PEOPLE:
                where = selection;
                break;
            default:
                throw new IllegalArgumentException("UnKnow URI: " + uri);
        }
        return where;
    }

    private void notifyChange(@NonNull Uri uri) {
        if (getContext() != null) {
            getContext().getContentResolver().notifyChange(uri, null);
        }
    }
}
```

这里区分了操作一条还是多条数据，不同的Uri其实就是判断条件不一样。下面依次描述如何操作数据库：

* UriMatcher初始化：静态初始化，把能匹配的Uri都加入到改类中，方便判断；
* onCreate: 通过Grrendao拿到SQLiteDatabase，由于这里边更多接近原生代码，所以就使用SQLiteDatabase来操作，就不用Greendao的方法来操作了。
* getType：根据Uri来判断是哪种类型；
* 增删改查：完全是调用db的原生方法，简单的操作表方法，这里也不再介绍了。
* notifyChange：这个的目的是在于回调告诉那些监听了这个ContentProvider的组件，哪个Uri发生了改变；这里也不再介绍。

到这里自定义ContentProvider就已经实现了，而数据访问者如何使用请查看demo。

## Demo源码

[Service Demo](https://github.com/arvinljw/AndroidArt/tree/master/app/src/main/java/net/arvin/androidart/service)

[Broadcast Demo](https://github.com/arvinljw/AndroidArt/tree/master/app/src/main/java/net/arvin/androidart/broadcast)

[ContentProvider Demo](https://github.com/arvinljw/AndroidArt/tree/master/app/src/main/java/net/arvin/androidart/provider)











