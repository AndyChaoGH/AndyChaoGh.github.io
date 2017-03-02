---
layout: post
title: Android学习感悟之Intent
author: arvinljw
---

本篇包括Intent、Intent过滤器和常见的Intent

# 简介

Intent在开发过程中，有着举足轻重的作用，最常见的就是界面间的跳转。对他的翻译中，我觉得“意图”解释的最到位，他用代码表示我们的想法，并让系统去执行。本篇内容包括Intent、Intent过滤器和常见的Intent，下面就依次解释。

## Intent

官方的解释：Intent是一个消息传递对象，您可以使用它从其他应用组件请求操作。在组件之间的通信主要包括：

* 启动 Activity

	 * startActivity(Intent intent)
	 
	 * startActivityForResult(Intent intent, int requestCode)

* 启动服务

	* startService(Intent service)

	* bindService(Intent service, ServiceConnection conn, int flags)

* 传递广播

	* sendBroadcast(Intent intent)

Intent又分为显示Intent和隐式Intent。

显示是指指定了明确的要启动的组建，即设置了ComponentName，可以使用 setComponent()、setClass()、setClassName() 或 Intent 构造函数设置组件名称；

隐式则就是不指定特定的组件，通过申明一些要执行的操作，让其它组件来处理，这里就设计到了Intent过滤器，用来匹配Intent是否有组件去处理，具体匹配规则在下文再分析。

**注意：5.0以后如果使用隐式 Intent 调用 bindService()，系统会引发异常。因为您无法确定哪些服务将响应 Intent，且用户无法看到哪些服务已启动。**

Intent包含了：ComponentName、Action、Data、Category、Extra、Flag这几个部分，都提供了不同的方法去设置，不外乎都是setXx或者addXx方法。

使用隐式选择器时，如果系统中有多个能匹配上的组件就会通过一个对话框去选择用哪个组件去打开。由于系统对这种默认的对话框提供了记住选择的功能，所以下次再想选择就不行了，就像分享功能，这时候就可以使用Intent.createChooser(Intent target, CharSequence title)去强制打开对话框。

**注意：由于隐式Intent可能匹配不到相应的组件，所以我们使用时最好加上一个判断intent.resolveActivity(getPackageManager()) != null，不为空表示能找到目标组件**

## Intent过滤器

隐式Intent，系统是如何找到对应的组件的呢？其实就是这些组件再Manifest文件中定义时，加上了\<intent-filter>标签，当然可以不定义或者定义一个或者定义多个该标签，其中包括了\<action>、\<category>和\<data>三个标签，三个标签也是可以不定义的。

\<action> 标签，只需设置name属性，内容可自定义，字符串；

\<data> 标签使用一个或多个指定数据 URI 各个方面（scheme、host、port、path 等）和 MIME 类型的属性，声明接受的数据类型；

\<category> 标签，也是设置name属性，内容可自定义，字符串。

对于action，只要组件中某一个intent-filter中的某一个action匹配上了即通过；若没有action标签，则只有当intent中也没有设置action才算通过；

对于data，比较复杂，它包含URI结构和数据类型，首先URI的几个属性存在着以下的依赖关系，

URI结构：\<scheme>://\<host>:\<port>/\<path>

* 如果未指定scheme，则会忽略host。
* 如果未指定host，则会忽略port。
* 如果未指定scheme和host，则会忽略path。

Intent中的URI与过滤器中的URI规范进行比较时，它仅与过滤器中包含的部分 URI 进行比较。所以只要Intent中包含了过滤器中的URI则URI匹配通过；其中path的匹配可包含通配符(*)

URI结构和MIME类型，匹配时，需要分为四种情况，都无、有其一或者都有：

* 仅当过滤器未指定任何 URI 或 MIME 类型时，不含 URI 和 MIME 类型的 Intent 才会通过测试。

* 对于包含 URI 但不含 MIME 类型（既未显式声明，也无法通过 URI 推断得出）的 Intent，仅当其 URI 与过滤器的 URI 格式匹配、且过滤器同样未指定 MIME 类型时，才会通过测试。

* 仅当过滤器列出相同的 MIME 类型且未指定 URI 格式时，包含 MIME 类型、但不含 URI 的 Intent 才会通过测试。

* 仅当 MIME 类型与过滤器中列出的类型匹配时，同时包含 URI 类型和 MIME 类型的 Intent 才会通过测试的 MIME 类型部分。如果过滤器只是列出 MIME 类型，则假定组件支持 content: 和 file: 数据。

对于category，intent可以不设置，也可通过，但是若是设置了，则intent的每个类别均必须与过滤器中的类别匹配才能通过，而在过滤器中，必须要写上默认的属性值：android.intent.category.DEFAULT；

综上，当每一项都匹配通过时即可找到对应的组件。

## 常用的Intent

1、发送短信／彩信

```
public void composeMmsMessage(String message, Uri attachment) {
    Intent intent = new Intent(Intent.ACTION_SEND);
    intent.setData(Uri.parse("smsto:"));  
    intent.putExtra("sms_body", message);
    intent.putExtra(Intent.EXTRA_STREAM, attachment);//附加的图像或视频的 Uri
    if (intent.resolveActivity(getPackageManager()) != null) {
        startActivity(intent);
    }
}
```

2、打开系统设置

```
public void openSettings() {
    Intent intent = new Intent(Intent.ACTION_SETTINGS);
    if (intent.resolveActivity(getPackageManager()) != null) {
        startActivity(intent);
    }
}
```

3、打开电话应用

```
public void dialPhoneNumber(String phoneNumber) {
    Intent intent = new Intent(Intent.ACTION_DIAL);
    intent.setData(Uri.parse("tel:" + phoneNumber));
    if (intent.resolveActivity(getPackageManager()) != null) {
        startActivity(intent);
    }
}
```

4、打开浏览器浏览网址

```
/**
* @param url 需要以http或https开头
*/
public static void toHtml(Context context, String url) {
	Intent intent = new Intent(Intent.ACTION_VIEW);
	intent.setData(Uri.parse(url));
	context.startActivity(intent);
}
```

5、打开系统日历

```
/**
* 打开系统日历
*/
public static void callCalendar(Context context) {
	Intent intent = new Intent(Intent.ACTION_VIEW);
	intent.setData(Uri.parse("content://com.android.calendar/time"));
	context.startActivity(intent);
}
```

6、打开系统计算器

```
@TargetApi(15)
public static void callCalculator(Context context) {
	ArrayList<HashMap<String, Object>> items = new ArrayList<HashMap<String, Object>>();
	final PackageManager pm = context.getPackageManager();
	List<PackageInfo> packs = pm.getInstalledPackages(0);
	for (PackageInfo pi : packs) {
		if (pi.packageName.toLowerCase().contains("calcul")) {
			HashMap<String, Object> map = new HashMap<String, Object>();
			map.put("appName", pi.applicationInfo.loadLabel(pm));
			map.put("packageName", pi.packageName);
			items.add(map);
		}
	}
	if (items.size() >= 1) {
		String packageName = (String) items.get(0).get("packageName");
		Intent i = pm.getLaunchIntentForPackage(packageName);
		if (i != null)
			context.startActivity(i);
	} else {
		Toast.makeText(context, "没有安装计算器", Toast.LENGTH_SHORT).show();
	}
}
```

## 参考

[Intent 和 Intent 过滤器](https://developer.android.com/guide/components/intents-filters.html)

[通用Intent](https://developer.android.com/guide/components/intents-common.html)

## 隐式Intent匹配测试代码

[隐式Intent匹配测试代码含总结](https://github.com/arvinljw/AndroidArt/tree/master/app/src/main/java/net/arvin/androidart/intent)

















