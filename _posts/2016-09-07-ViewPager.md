---
layout: post
title: ViewPager（一屏多页、无限滑动、自动切换）
author: arvinljw
---

前段时间在腾讯视频中看到一个效果，是一个广告轮播，然后一屏还显示了多页，然后就自己实现了一下。

## 原理 

国际惯例先上效果图:

![](http://img.blog.csdn.net/20160831102856251?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQv/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

实现如上效果需要两个功能：一屏多页、无限滑动、自动切换，下边将分别简单的介绍其原理。

1、一屏多页

不限制子View在其范围内。

2、无限滑动

限于文字功底不足，所以举例说明一下。有一个ViewPager，有5页数据{1,2,3,4,5}，那么在1前加一页为0，内容和5一样，同时在5后加一页为6，内容和1一样（Adapter中体现）；然后在页面切换后判断，如果是0页时跳转到5页，6页时跳转到1页。

3、自动切换

每隔一段时间，切换到下一页

## 实现

（1）让ViewPager的父容器具有Android:clipChildren="false"和android:layerType="software"这两个属性，前一句主要意思是不限制子View在其范围内，后一句是启动硬件加速。

（2）ViewPager设置marginLeft和marginRight属性，这两个大小会分别导致左右两边的page显示的大小。
 
```
<?xml version="1.0" encoding="utf-8"?>  
<RelativeLayout xmlns:android="http://schemas.android.com/apk/res/android"  
    xmlns:tools="http://schemas.android.com/tools"  
    android:layout_width="match_parent"  
    android:layout_height="match_parent"  
    android:clipChildren="false"  
    android:layerType="software"  
    tools:context="net.arvin.viewpagertransdemo.MainActivity">  
  
    <android.support.v4.view.ViewPager  
        android:id="@+id/mPager"  
        android:layout_width="match_parent"  
        android:layout_height="200dp"  
        android:layout_marginLeft="32dp"  
        android:layout_marginRight="32dp" />  
</RelativeLayout>
```
（3）创建可循环的Adapter，如下：

```
import android.support.v4.app.Fragment;  
import android.support.v4.app.FragmentManager;  
import android.support.v4.app.FragmentStatePagerAdapter;  
import net.arvin.viewpagertransdemo.SimpleFragment;  
  
import java.util.List;  
  
/** 
 * Created by arvin on 2016/8/24 09:52 
 */  
public class LoopPagerAdapter extends FragmentStatePagerAdapter {  
    private List<Integer> items;  
  
    public LoopPagerAdapter(FragmentManager fm, List<Integer> items) {  
        super(fm);  
        this.items = items;  
    }  
  
    @Override  
    public int getCount() {  
        //比原来的页数多两页，因为在第一页前加；最后一页后也要加一页  
        return items.size() + 2;  
    }  
  
    @Override  
    public Fragment getItem(int position) {  
        //将position转化为对应在items的position  
        if (position == items.size() + 1) {  
            //在最后一页的时候显示第一页的内容，  
            position = 0;  
        } else if (position == 0) {  
            //在第一页的时候显示最后一页的内容  
            position = items.size() - 1;  
        } else {  
            position -= 1;  
        }  
        //显示相应页的内容  
        return new SimpleFragment(items.get(position));  
    }  
}  
```
其中SimpleFragment就是作为背景显示传入的颜色值，就不解释了。

（4）设置ViewPager，如下：

```
mPager = (ViewPager) findViewById(R.id.mPager);  
mPager.setAdapter(new LoopPagerAdapter(getSupportFragmentManager(), mItems));  
mPager.setPageMargin(Utils.dp2px(1));  
mPager.setOffscreenPageLimit(mItems.size());  
mPager.setCurrentItem(currentPosition);  
mPager.setOnPageChangeListener(this);  
```
setPageMargin这是设置页与页之间的距离；

currentPosition初始化为1，这才是真是的第一页的数据；

其他的都是ViewPager的基本东西，这里就不解释了；

（5）处理页面切换后的逻辑，如下：

```
@Override  
public void onPageSelected(int position) {  
    currentPosition = position;  
    if (position == getColors().size() + 1) {  
        currentPosition = 1;  
        isNeedChange = true;  
    } else if (position == 0) {  
        isNeedChange = true;  
        currentPosition = getColors().size();  
    } else {  
        isNeedChange = false;  
    }  
}  
  
@Override  
public void onPageScrollStateChanged(int state) {  
    if (state == ViewPager.SCROLL_STATE_IDLE && isNeedChange) {  
        mPager.setCurrentItem(currentPosition, false);  
    }  
} 
```
首先，isNeedChange表示是否需要改变界面，也就是切换页面后position为第一页或者最后一页时就需要改变，其他时候就不需要，因为处于所有页面的中间；
然后，currentPosition表示切换后的真实位置，上边的代码逻辑也很简单，介绍原理时已说明；

最后，在onPageScrollStateChanged中在停止滚动且可以改变界面时，我们就跳转到真实的页面，且不加动画。

（6）自动切换

自动切换，每隔一段时间触发一个事件，可以用Timer，用法也很简单，就不介绍了，因为我们用的不是它；这里使用Handler去实现，然而Handler的使用往往会引起内存泄露，所以我们需要封装一下，使用弱引用去处理一下，这里也就不详细介绍了，直接上Handler的代码：

```
public class WeakHandler extends Handler {  
    private WeakReference<IWeakHandler> mCallback;  
  
    public WeakHandler(IWeakHandler activity) {  
        this.mCallback = new WeakReference<>(activity);  
    }  
  
    public WeakHandler(Looper looper,IWeakHandler activity){  
        super(looper);  
        this.mCallback = new WeakReference<>(activity);  
    }  
  
    @Override  
    public void handleMessage(Message msg) {  
        super.handleMessage(msg);  
        if (mCallback != null) {  
            mCallback.get().handleMessage(msg);  
        }  
    }  
}  
```
然后我们在主界面中，首先初始化一下这个WeakHandler：

```
mHandler = new WeakHandler(this);  
```
这里需要实现一个IweakHandler的接收到消息的回调，收到消息后就应该切换界面，并再次发出一个自动切换的消息：

```
@Override  
public void handleMessage(Message msg) {  
    if (msg.what == TRANS_ANIM_CODE) {  
        mPager.setCurrentItem(++currentPosition, true);  
        startTrans();  
    }  
} 
```
然后我们应该再哪里去发出或者移除自动切换的消息，没错就分别在界面可见和不可见时就行了：

```
@Override  
protected void onResume() {  
    super.onResume();  
    startTrans();  
}  
  
@Override  
protected void onPause() {  
    super.onPause();  
    stopTrans();  
}  
  
private void startTrans() {  
    mHandler.sendEmptyMessageDelayed(TRANS_ANIM_CODE, TRANS_ANIM_DURING);  
}  
  
private void stopTrans() {  
    mHandler.removeMessages(TRANS_ANIM_CODE);  
}  
```
这样我们就实现了自动切换了，这时候你可能会发现一个体验上的问题，当用户正在操作时，这个收到这个切换消息时，也毅然决然的切换了，让用户觉得怎么回事，我还想慢慢的看会呢，接下来就来处理这个问题。

（7）在用户操作时不自动切换

正如这个小标题所说，那要怎么做才能不自动切换呢，对，就是在用户触摸到Page的时候就移除掉自动切换的消息，在用户没有触摸到Page的时候就发出自动切换的消息。这时候就需要我们去重写ViewPager，监听触摸事件，并给出相应的回调，代码也很简单，如下：

```
public class UserOperateCallbackViewPager extends ViewPager {  
    private IUserOperating mUserOperating;  
  
    public UserOperateCallbackViewPager(Context context) {  
        super(context);  
    }  
  
    public UserOperateCallbackViewPager(Context context, AttributeSet attrs) {  
        super(context, attrs);  
    }  
  
    public void setUserOperating(IUserOperating userOperating) {  
        this.mUserOperating = userOperating;  
    }  
  
    @Override  
    public boolean onTouchEvent(MotionEvent ev) {  
        switch (ev.getAction()) {  
            case MotionEvent.ACTION_DOWN:  
                if (mUserOperating != null) {  
                    mUserOperating.userOperating(true);  
                }  
                break;  
            case MotionEvent.ACTION_UP:  
                if (mUserOperating != null) {  
                    mUserOperating.userOperating(false);  
                }  
                break;  
        }  
        return super.onTouchEvent(ev);  
    }  
  
    public interface IUserOperating {  
        void userOperating(boolean isOperating);  
    }  
}  
```
然后我们就用UserOperateCallbackViewPager去替换掉原生的ViewPager，再为新的ViewPager设置一个回调，并处理回调即可：

```
mPager.setUserOperating(this);  
  
@Override  
public void userOperating(boolean isOperating) {  
    if (isOperating) {  
        stopTrans();  
    } else {  
        startTrans();  
    }  
}  
```
这样我们的自动切换的功能就正式完成了.

我们只需要调整布局放到我们的项目中，就能实现比较优雅的界面了。

## 源码

**[源码链接](https://github.com/arvinljw/ViewpagerTransDemo)**

