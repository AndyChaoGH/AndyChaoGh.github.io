---
layout: post
title: Android学习感悟之Activity
author: arvinljw
---

Android学习感悟之Activity,包含生命周期、启动模式以及一些Activity中常用的方法。

# 序

在Android开发中，用的最为平凡的，最不能少的就是Activity了，所以作为学习感悟的第一篇我觉得再适合不过了。还记得刚接触Android的时候，第一件事就是找到Hello World是怎么显示到手机上的，那就是我第一次接触到Activity。话不多说，直接进入正题。

# Activity的生命周期

Activity，我们常常把它译为活动，大家都知道任何一件事都有他的流程，而在Android中把他叫做生命周期，一件事的流程无外乎都会包含，开始，经过，结束，而在Android中Activity同样有这些过程，只不过在程序中是由系统去回调这些步骤的，下边我们可以先看一下，这些过程有哪些。
来自于Android developer training：

![](../images/basic-lifecycle.png)

可以看到依次包含:onCreate()->onStart()->onResume()->onPaused()->onStop()->onDestroy()

Activity的开始：onCreate->onStart()->onResume()；

Activity的结束：onPaused()->onStop()->onDestroy；

Activity的经过：是由界面可视化后，与用户交互的过程。这里边涉及到一个名词——可视化，意思很简单，就是由界面不可见到可见的这个过程。

## 正常情况下Activity生命周期

这个图其实已经包含了所有正常情况下Activity生命周期的内容。而正常的生命周期的回调也有很多种情况，接下来会逐一介绍。**（注：接下来说的启动都是指第一次启动和标准模式的启动方式）**

### 1、正常的启动和关闭。

**（1）启动Activity A**，系统就会调用onCreate，创建这个Activity，创建成功后会马上调用onStart，这时候Activity就已经可见了，但还没有出现在在前台，还无法交互，紧接着会调用onResume，这时候界面就已经可见了，这时候这一次启动就已经走完了；

**（2）结束Activity A**，会先调用onPause，调用完成后界面就不分不可见了，紧接着这时候会掉用onStop，调用完成后就完全看不见了，最后就会调用onDestroy去销毁这个Activity A，这时候这个Activity就结束了。

### 2、启动Activity A后再从A启动和结束Activity B过程中，Activity A的生命周期的回调。

**（1）当A启动好后，从A启动B**，这时候A很明显会被B遮住，所以从视觉上我们应该可以猜到应该会调用onPause和onStop，实际上也确实是这样；

**（2）结束B**，这时候A就应该重新可见，系统便会给我们依次回调onRestart->onSatrt->onResume方法，这时候A的界面又重新可见了。

**（3）注意：**这里边有两个知识点比较特殊，（1）如果B的主题被设置被背景透明，那么从A启动B时，A不会调用onStop方法，具体原因暂时还未深究；（2）A的onPause和B的onResume谁先调用，从源码中可以得到答案，应该是A的onPause先调用，所以为了用户体验，我们不应该在onPause中做耗时的操作。

### 3、启动Activity A后再从A打开和关闭Dialog过程中，Activity A的生命周期的回调。

**（1）A启动好后，从A打开Dialog**，这时候A便会被遮住一部分，系统会调用A的onPause方法；

**（2）关闭Dialog**，这时候A重新全部可见，会调用onResume方法。

## 异常情况下的生命周期

上边说到正常的生命周期会有那么几种情况，那什么叫不正常的呢？其实就是Activity在资源相关的系统配置发生改变或者系统内存不足时被杀死，并重新创建的时候，这时候就不大相同了。接下来分析这两种情况。

### 1、资源以及相关配置发生改变，被销毁并重建

资源加载机制比较复杂，我暂时还没有太多了解，这里也就不过多解释；举个出现这种情况的例子吧，比如程序有两张图片对应横屏和竖屏时加载，当Activity从竖屏转到横屏时，由于系统配置发生改变，在默认情况下，Activity就会被销毁并重建，当然我们也可以阻止重建该Activity。

（1）在这种情况下Activity的销毁依然还是会调用onPause，onStop，onDestroy方法，同时还会调用onSaveInstanceState，通过参数Bundle来保存当前Activity的状态，该方法的回调早onStop之前，和onPause没有固定的顺序；重建的时候除了会调用onCreate，onStart，onResume以外还会多调用一个onRestoreInstanceState方法，通过参数Bundle来恢复状态，该方法在onStart之后调用。值得注意的是，系统本身在这两个方法里已经为我们保存了一些数据，例如View的数据，通过查看源码我们可以找到在View中也有onSaveInstanceState和onRestoreInstanceState方法，用来保存和恢复View中的一些数据，而这个过程是由Activity通知Window，window通知decorView，decorView再一一通知子View来完成的，通知的过程和View的绘制和事件回调流程一致，这是一种委托的思想。

### 2、资源内存不足导致低优先级的Activity被销毁。

这种情况不好模拟，但是他的数据恢复和第一种情况是完全一致的，而这里描述一下Activity的优先级，从高到低分为：

（1）前台Activity——正在交互的

（2）可见但非前台——例如弹出一个Dialog，无法直接交互的

（3）后台Activity——被暂停的，执行了onStop方法的

### 3、阻止配置改变时被销毁重建的方法

我们可以在AndroidManifest.xml文件中给Activity指定configChanges属性，并根据不同情况赋予不同的值即可保证改变时不被销毁重建，下边就举几个例子来说明：

**（1）**当我们只考虑屏幕旋转引起的重启时，我们只保留竖屏或者横屏，这样就让旋转时不改变布局方向来保证不被销毁重启，我们可以给Activity指定screenOrientation属性，属性值如下：

```
android:screenOrientation="portrait"//保留竖屏
android:screenOrientation="landscape"//保留横屏
```
该属性涉及到横竖屏的属性值就有好多个，其具体含义如下：

| Value | Meaning |
| :-----: | ------- |
| "unspecified" | 默认值 由系统来判断显示方向.判定的策略是和设备相关的，所以不同的设备会有不同的显示方向 |
| "landscape" | 	横屏显示（宽比高要长） |
| "portrait" | 竖屏显示(高比宽要长)  |
| "user" | 用户当前首选的方向 |
| "behind" | 和该Activity下面的那个Activity的方向一致(在Activity堆栈中的) |
| "sensor" | 有物理的感应器来决定。如果用户旋转设备这屏幕会横竖屏切换 |
| "nosensor" | 忽略物理感应器，这样就不会随着用户旋转设备而更改了 （ "unspecified"设置除外 ） |

**（2）**还有一种就是使用configChanges属性，并设置为：

```
 android:configChanges="orientation"
```

同时设置多个属性值时，只需用“｜”隔开即可。下边还是看一下该属性常用的属性值的含义吧：如下：

| Value | Meaning |
| :-----: | ------- |
| "orientation" | 屏幕方向发生了改变，例如旋转手机屏幕 |
| "locale" | 本地位置放生改变，一般为切换了系统语言 |
| "keyboardHidden" | 键盘的可访问性发生了改变，例如用户调出了键盘  |
| "screenSize" | 当屏幕尺寸发生改变，例如屏幕旋转也会导致屏幕尺寸发生改变，若minSdkVersion或者targetSdkVersion任一大于等于13，则就会引起重启 |
| "smallestScreenSize" | 当接入外接显示器时，若minSdkVersion或者targetSdkVersion任一大于等于13，则就会引起重启 |

## Activity的启动模式

### 1、启动模式介绍

Android系统为了保证资源不浪费，尽可能的不要创建多余的Activity，所以引入了启动模式，Actvity的启动模式包含：**standard、singleTop、singleTask、singleInstance**。说到启动模式，必然就会涉及到任务栈的问题，这是系统为我们提供的管理Activity的一个结构，栈的特点就是先进后出。那Activity创建时是怎么进栈的呢，这个就要看不同的启动模式，来区别对待了。

**（1）standard：标准模式，这是系统默认的启动模式。**每次启动一个Activity都会重新创建一个Activity实例，不管这个Activity是否已经存在实例。这时候被启动的Activity进入的栈就是启动它的Activity所在的任务栈。当我们使用ApplicationContext去启动standard模式的Activity时是会报错的，而ApplicationContext并没有任务栈。解决办法是为要启动的Activity指定一个FLAG_ACTIVITY_NEW_TASK标记位，这时就能为新创建的Activity创建一个新的任务栈。

**（2）SingleTop：栈顶复用模式。**如果新的Activity的实例已经位于栈顶，则不会被重新创建，只会回调onNewIntent方法，便可拿到当前请求的信息。若不再栈顶，则依然会重新创建一个实例。

**（3）singleTask：栈内复用模式。**这其实是一种单例模式，只要该Activity所需的任务栈中存在其实例，便不会重新创建该实例，只会回调onNewIntent方法，而在该Activity上的Activity会被移除掉，从而使该Activity移动到栈顶。若不存在，便会在该Activity所需的任务栈中创建实例，若其所需的任务栈不存在，则会先创建该任务栈。但是使用ApplicationContext去启动一个SingleTask模式的Activity依旧会报错。其中说到了Activity所需的任务栈，这个就是要使用TaskAffinity属性来设定，默认任务栈的名字都是包名，这个属性只有和singleTask一起使用才有效。

**（4）singleInstance：单实例模式。**这个就是加强版的singleTask，他只会单独存在于一个任务栈中。

### 2、启动模式的应用场景

| Mode | Use Case |
| :----: | -------- |
| standard | 默认都是使用该模式 |
| singleTop | 适合接受通知并启动的内容展示页面，好比新闻详情界面，收到10个新闻推送，每个都要打开新闻详情，若打开十次那肯定是不合适的。  |
| singleTask | 适合作为程序入口点，例如浏览器的主界面。不管从多少个应用启动浏览器，只会启动主界面一次，其余情况都会走onNewIntent，并且会清空主界面上面的其他页面。 |
| singleInstance | 适合需要与程序分离开的页面。例如闹铃提醒，将闹铃提醒与闹铃设置分离。singleInstance不要用于中间页面，如果用于中间页面，跳转会有问题，比如：A -> B (singleInstance) -> C，完全退出后，在此启动，首先打开的是B。 |

## Activity中常用的方法

```
1、生命周期中的各个方法；
2、finish()－>关闭Activity；
3、onBackPressed()->按返回键回调，默认实现是finish掉Activity；
4、startActivity(...)－>分别表示启动Activity；
5、startActivityForResult(...)->启动Activity并能接受启动的那个Activity回传的参数，接收参数使用onActivityResult(...)方法；
6、overridePendingTransition(...)->启动Activity的启动动画，必须紧跟在startActivity()或者finish()的后边才有效；
7、runOnUiThread(...)->在主线程中运行，通常在子线程中想要更新界面时使用，内部使用Handler实现；
8、getResource()－>获取资源,方便获取图片、颜色、尺寸、文本等资源；
9、registerReceiver(...)以及unregisterReceiver(...)->注册广播以及注销广播；
10、bindService()以及unbindService()->绑定服务以及解绑服务；
11、getContentResolver()->用于获取contentProvider数据。
```

## 总结

写到这里基本上Activity的生命周期和启动模式就介绍的差不多了，如果说还差什么的话，我觉得就应该是Activity的标志未以及IntentFilter的匹配规则，这两个内容留到Intent中再写吧，这篇感觉已经太多了。如果有任何不对的地方，请发我邮件，感谢～


## 参考资料

《Android开发艺术探究》、Android developer training。
