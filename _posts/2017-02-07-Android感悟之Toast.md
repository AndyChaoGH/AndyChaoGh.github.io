---
layout: post
title: Android学习感悟之Toast
author: arvinljw
---

本篇包括Toast实现以及如何自定义Toast内容

# Toast的实现

```
Toast.makeText(context, message, Toast.LENGTH_SHORT).show();
```

这样的一句话是我们显示Toast最常见的视线方式，接下来我们就从这句话开始研究。

先看makeText方法的实现：

```
	/**
     * Make a standard toast that just contains a text view.
     *
     * @param context  The context to use.  Usually your {@link android.app.Application}
     *                 or {@link android.app.Activity} object.
     * @param text     The text to show.  Can be formatted text.
     * @param duration How long to display the message.  Either {@link #LENGTH_SHORT} or
     *                 {@link #LENGTH_LONG}
     *
     */
    public static Toast makeText(Context context, CharSequence text, @Duration int duration) {
        Toast result = new Toast(context);

        LayoutInflater inflate = (LayoutInflater) context.getSystemService(Context.LAYOUT_INFLATER_SERVICE);
        View v = inflate.inflate(com.android.internal.R.layout.transient_notification, null);
        TextView tv = (TextView)v.findViewById(com.android.internal.R.id.message);
        tv.setText(text);
        
        result.mNextView = v;
        result.mDuration = duration;

        return result;
    }
    
//transient_notification.xml    
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:orientation="vertical"
    android:background="?android:attr/toastFrameBackground">

    <TextView
        android:id="@android:id/message"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:layout_weight="1"
        android:layout_gravity="center_horizontal"
        android:textAppearance="@style/TextAppearance.Toast"
        android:textColor="@color/bright_foreground_dark"
        android:shadowColor="#BB000000"
        android:shadowRadius="2.75"
        />

</LinearLayout>

```

整体来看，比较简单就是new一个Toast对象，然后设置它要显示的View，以及显示时间，然后把这个对象返回，最后再调用show()，显示出来。

接下来更深入一点看看，Toast初始化时做了什么？

```
	public Toast(Context context) {
        mContext = context;
        mTN = new TN();
        mTN.mY = context.getResources().getDimensionPixelSize(
                com.android.internal.R.dimen.toast_y_offset);
        mTN.mGravity = context.getResources().getInteger(
                com.android.internal.R.integer.config_toastDefaultGravity);
    }
    
    //引用资源
    <dimen name="toast_y_offset">64dip</dimen>
    
    <!-- Default Gravity setting for the system Toast view. Equivalent to: Gravity.CENTER_HORIZONTAL | Gravity.BOTTOM -->
    <integer name="config_toastDefaultGravity">0x00000051</integer>
```

看起来也不难，也是初始化一些值，可以看到的是，Y轴偏移为64dp，相对位置底部且水平居中，还需要细看的就是TN的初始化，TN到底是什么呢？接着看。

```
private static class TN extends ITransientNotification.Stub {
	final Runnable mHide = new Runnable() {
		@Override
		public void run() {
			handleHide();
			// Don't do this in handleHide() because it is also invoked by handleShow()
			mNextView = null;
			}
		};

	private final WindowManager.LayoutParams mParams = new WindowManager.LayoutParams();
	final Handler mHandler = new Handler() {
		@Override
		public void handleMessage(Message msg) {
			IBinder token = (IBinder) msg.obj;
			handleShow(token);
		}
	};
	
	int mGravity;
	int mX, mY;
	float mHorizontalMargin;
	float mVerticalMargin;
	View mView;
	View mNextView;
	int mDuration;
	WindowManager mWM;

	static final long SHORT_DURATION_TIMEOUT = 5000;
	static final long LONG_DURATION_TIMEOUT = 1000;

	TN() {
            // XXX This should be changed to use a Dialog, with a Theme.Toast
            // defined that sets up the layout params appropriately.
            final WindowManager.LayoutParams params = mParams;
            params.height = WindowManager.LayoutParams.WRAP_CONTENT;
            params.width = WindowManager.LayoutParams.WRAP_CONTENT;
            params.format = PixelFormat.TRANSLUCENT;
            params.windowAnimations = com.android.internal.R.style.Animation_Toast;
            params.type = WindowManager.LayoutParams.TYPE_TOAST;
            params.setTitle("Toast");
            params.flags = WindowManager.LayoutParams.FLAG_KEEP_SCREEN_ON
                    | WindowManager.LayoutParams.FLAG_NOT_FOCUSABLE
                    | WindowManager.LayoutParams.FLAG_NOT_TOUCHABLE;
        }
        
	@Override
	public void show(IBinder windowToken) {
		if (localLOGV) Log.v(TAG, "SHOW: " + this);
		mHandler.obtainMessage(0, windowToken).sendToTarget();
	}

	@Override
	public void hide() {
		if (localLOGV) Log.v(TAG, "HIDE: " + this);
		mHandler.post(mHide);
	}

	public void handleShow(IBinder windowToken) {
        if (localLOGV) Log.v(TAG, "HANDLE SHOW: " + this + " mView=" + mView
                    + " mNextView=" + mNextView);
        if (mView != mNextView) {
            // remove the old view if necessary
            handleHide();
            mView = mNextView;
            Context context = mView.getContext().getApplicationContext();
            String packageName = mView.getContext().getOpPackageName();
            if (context == null) {
                context = mView.getContext();
            }
            mWM = (WindowManager)context.getSystemService(Context.WINDOW_SERVICE);
            // We can resolve the Gravity here by using the Locale for getting
            // the layout direction
            final Configuration config = mView.getContext().getResources().getConfiguration();
            final int gravity = Gravity.getAbsoluteGravity(mGravity, config.getLayoutDirection());
            mParams.gravity = gravity;
            if ((gravity & Gravity.HORIZONTAL_GRAVITY_MASK) == Gravity.FILL_HORIZONTAL) {
                mParams.horizontalWeight = 1.0f;
            }
            if ((gravity & Gravity.VERTICAL_GRAVITY_MASK) == Gravity.FILL_VERTICAL) {
                mParams.verticalWeight = 1.0f;
            }
            mParams.x = mX;
            mParams.y = mY;
            mParams.verticalMargin = mVerticalMargin;
            mParams.horizontalMargin = mHorizontalMargin;
            mParams.packageName = packageName;
            mParams.hideTimeoutMilliseconds = mDuration ==
                    Toast.LENGTH_LONG ? LONG_DURATION_TIMEOUT : SHORT_DURATION_TIMEOUT;
            mParams.token = windowToken;
            if (mView.getParent() != null) {
                if (localLOGV) Log.v(TAG, "REMOVE! " + mView + " in " + this);
                mWM.removeView(mView);
            }
            if (localLOGV) Log.v(TAG, "ADD! " + mView + " in " + this);
            mWM.addView(mView, mParams);
            trySendAccessibilityEvent();
        }
    }

	private void trySendAccessibilityEvent() {
        AccessibilityManager accessibilityManager =
                    AccessibilityManager.getInstance(mView.getContext());
        if (!accessibilityManager.isEnabled()) {
            return;
        }
        // treat toasts as notifications since they are used to
        // announce a transient piece of information to the user
        AccessibilityEvent event = AccessibilityEvent.obtain(
                AccessibilityEvent.TYPE_NOTIFICATION_STATE_CHANGED);
        event.setClassName(getClass().getName());
        event.setPackageName(mView.getContext().getPackageName());
        mView.dispatchPopulateAccessibilityEvent(event);
        accessibilityManager.sendAccessibilityEvent(event);
    }        

    public void handleHide() {
        if (localLOGV) Log.v(TAG, "HANDLE HIDE: " + this + " mView=" + mView);
        if (mView != null) {
            // note: checking parent() just to make sure the view has
            // been added...  i have seen cases where we get here when
            // the view isn't yet added, so let's try not to crash.
            if (mView.getParent() != null) {
                if (localLOGV) Log.v(TAG, "REMOVE! " + mView + " in " + this);
                mWM.removeViewImmediate(mView);
            }
            mView = null;
        }
    }
}
```

这个代码中我们能知道，他其实就是初始化mParams，而这个属性是WindowManager.LayoutParams，从而可以猜到估计Toast的View是要加到Window上。而且TN继承自ITransientNotification.Stub，用于进程间通信,所以TN类的具体作用应该是Toast类的回调对象，其他进程通过调用TN类的具体对象来操作Toast的显示和消失。ITransientNotification.Stub的源码为：

```
oneway interface ITransientNotification {
    void show(IBinder windowToken);
    void hide();
}
```

其具体实现在源码中也体现了，其实就是通过handler去发消息。这里的Handler是在当前线程初始化的，也就是说如果在非主线程试用Toast的话，则需要先调用Looper.perpare()才不会报错。

如果show被调用，则最终会调用handleShow()，不难看出这就是一个Window addView的过程。

如果hide被调用，则会调用handleHide()，也很简单，就是Window removeView的过程。

这样就可以得出，showToast必须要在主线程操作，因为window的addView和removeView必须要在主线程。

接下来我们就需要看看TN是如何被回调的，接着Toast的show()，如下：

```
	public void show() {
        if (mNextView == null) {
            throw new RuntimeException("setView must have been called");
        }

        INotificationManager service = getService();
        String pkg = mContext.getOpPackageName();
        TN tn = mTN;
        tn.mNextView = mNextView;

        try {
            service.enqueueToast(pkg, tn, mDuration);
        } catch (RemoteException e) {
            // Empty
        }
    }
    
    private static INotificationManager sService;

    static private INotificationManager getService() {
        if (sService != null) {
            return sService;
        }
        sService = INotificationManager.Stub.asInterface(ServiceManager.getService("notification"));
        return sService;
    }
```

首先可以看到如果自定义的话，一定要设置Toast.mNextView，否则就会抛异常，接着看getService()，得到INotificationManager服务后，调用了enqueueToast方法将当前的Toast放入到系统的Toast队列中。这里INofiticationManager接口的具体实现类是NotificationManagerService类，主要看一下enqueueToast的实现：

```
	private final IBinder mService = new INotificationManager.Stub() {
        
        @Override
        public void enqueueToast(String pkg, ITransientNotification callback, int duration)
        {
            if (DBG) {
                Slog.i(TAG, "enqueueToast pkg=" + pkg + " callback=" + callback
                        + " duration=" + duration);
            }
            if (pkg == null || callback == null) {
                Slog.e(TAG, "Not doing toast. pkg=" + pkg + " callback=" + callback);
                return ;
            }
            final boolean isSystemToast = isCallerSystem() || ("android".equals(pkg));
            final boolean isPackageSuspended =
                    isPackageSuspendedForUser(pkg, Binder.getCallingUid());
            if (ENABLE_BLOCKED_TOASTS && (!noteNotificationOp(pkg, Binder.getCallingUid())
                    || isPackageSuspended)) {
                if (!isSystemToast) {
                    Slog.e(TAG, "Suppressing toast from package " + pkg
                            + (isPackageSuspended
                                    ? " due to package suspended by administrator."
                                    : " by user request."));
                    return;
                }
            }
            synchronized (mToastQueue) {
                int callingPid = Binder.getCallingPid();
                long callingId = Binder.clearCallingIdentity();
                try {
                    ToastRecord record;
                    int index = indexOfToastLocked(pkg, callback);
                    // If it's already in the queue, we update it in place, we don't
                    // move it to the end of the queue.
                    if (index >= 0) {
                        record = mToastQueue.get(index);
                        record.update(duration);
                    } else {
                        // Limit the number of toasts that any given package except the android
                        // package can enqueue.  Prevents DOS attacks and deals with leaks.
                        if (!isSystemToast) {
                            int count = 0;
                            final int N = mToastQueue.size();
                            for (int i=0; i<N; i++) {
                                 final ToastRecord r = mToastQueue.get(i);
                                 //非系统Toast最多一次添加50条。
                                 if (r.pkg.equals(pkg)) {
                                     count++;
                                     if (count >= MAX_PACKAGE_NOTIFICATIONS) {
                                         Slog.e(TAG, "Package has already posted " + count
                                                + " toasts. Not showing more. Package=" + pkg);
                                         return;
                                     }
                                 }
                            }
                        }
                        //加入Toast队列
                        Binder token = new Binder();
                        mWindowManagerInternal.addWindowToken(token,
                                WindowManager.LayoutParams.TYPE_TOAST);
                        record = new ToastRecord(callingPid, pkg, callback, duration, token);
                        mToastQueue.add(record);
                        index = mToastQueue.size() - 1;
                        keepProcessAliveIfNeededLocked(callingPid);
                    }
                    // If it's at index 0, it's the current toast.  It doesn't matter if it's
                    // new or just been updated.  Call back and tell it to show itself.
                    // If the callback fails, this will remove it from the list, so don't
                    // assume that it's valid after this.
                    if (index == 0) {
                        showNextToastLocked();
                    }
                } finally {
                    Binder.restoreCallingIdentity(callingId);
                }
            }
        }
        
		...
	}
	
	void showNextToastLocked() {
        ToastRecord record = mToastQueue.get(0);
        while (record != null) {
            if (DBG) Slog.d(TAG, "Show pkg=" + record.pkg + " callback=" + record.callback);
            try {
                record.callback.show(record.token);
                scheduleTimeoutLocked(record);
                return;
            } catch (RemoteException e) {
                Slog.w(TAG, "Object died trying to show notification " + record.callback
                        + " in package " + record.pkg);
                // remove it from the list and let the process die
                int index = mToastQueue.indexOf(record);
                if (index >= 0) {
                    mToastQueue.remove(index);
                }
                keepProcessAliveIfNeededLocked(record.pid);
                if (mToastQueue.size() > 0) {
                    record = mToastQueue.get(0);
                } else {
                    record = null;
                }
            }
        }
    }
    
    
    static final int LONG_DELAY = PhoneWindowManager.TOAST_WINDOW_TIMEOUT  = 3500; // 3.5 seconds
    static final int SHORT_DELAY = 2000; // 2 seconds
    
    private void scheduleTimeoutLocked(ToastRecord r)
    {
        mHandler.removeCallbacksAndMessages(r);
        Message m = Message.obtain(mHandler, MESSAGE_TIMEOUT, r);
        long delay = r.duration == Toast.LENGTH_LONG ? LONG_DELAY : SHORT_DELAY;
        mHandler.sendMessageDelayed(m, delay);
    }
    
    private void handleTimeout(ToastRecord record)
    {
        if (DBG) Slog.d(TAG, "Timeout pkg=" + record.pkg + " callback=" + record.callback);
        synchronized (mToastQueue) {
            int index = indexOfToastLocked(record.pkg, record.callback);
            if (index >= 0) {
                cancelToastLocked(index);
            }
        }
    }
    
    void cancelToastLocked(int index) {
        ToastRecord record = mToastQueue.get(index);
        try {
            record.callback.hide();
        } catch (RemoteException e) {
            Slog.w(TAG, "Object died trying to hide notification " + record.callback
                    + " in package " + record.pkg);
            // don't worry about this, we're about to remove it from
            // the list anyway
        }
        ToastRecord lastToast = mToastQueue.remove(index);
        mWindowManagerInternal.removeWindowToken(lastToast.token, true);
        keepProcessAliveIfNeededLocked(record.pid);
        if (mToastQueue.size() > 0) {
            // Show the next one. If the callback fails, this will remove
            // it from the list, so don't assume that the list hasn't changed
            // after this point.
            showNextToastLocked();
        }
    }
```

就这样，从最开始的加入Toast队列，再到show，再到hide，源码依次如此，这样就是实现了Toast的显示和隐藏。

注意：Toast的显示时间只有两种，3.5s或者2s。

总结一下：整体过程就是新增一个Toast给系统，系统根据加入Toast队列的顺序，依次显示，再隐藏。

到这里Toast源码的具体实现的分析就结束了。

# 自定义Toast

根据源码分析来看，我们自定义Toast能改变的内容有Toast的显示内容和显示的位置。

很明显，内容的自定义就是设置一个自定义的View给Toast即可，位置就是改变Toast的Gravity属性即可，源码分析过后就发现很简单了，所以就不讲怎么做了，下面直接看几个截图就好了，然后我会把实现的源码放在下边。

顶部显示

![](../images/toast_top.png)

底部显示

![](../images/toast_bottom.png)

中间显示

![](../images/toast_center.png)


## 源码

[自定义Toast源码](https://github.com/arvinljw/AndroidArt/tree/master/app/src/main/java/net/arvin/androidart/Toast)



















