---
layout: post
title: Android学习感悟之View之二事件体系
author: arvinljw
---

本篇包括View的事件体系以及ViewGroup的事件体系

# 简介

本文包括：

* View的事件体系；
* ViewGroup的事件体系；

## View的事件体系

View的事件体系，其实和View的绘制是同样的思想，都是自上而下，逐一调用，同样View和ViewGroup的调用也是有一些差别，所以，下面也会分开来说明。

### View的事件体系

下边是我自己对源码理解后画的流程图：

![](../images/View事件传递.png)

View的事件流程中包括以下几个重要的事件：dispatchTouchEvent、onTouchListener、onTouchEvent、onLongClickListener、onClickListener；接下来就来解释一下如何调用的。

在用户点击某个View后，根据流程图我们可以看到，他是先调用dispatchTouchEvent方法，源码如下：

```
	public boolean dispatchTouchEvent(MotionEvent event) {
        boolean result = false;
        //...其它代码
        ListenerInfo li = mListenerInfo;
        if (li != null && li.mOnTouchListener != null
                && (mViewFlags & ENABLED_MASK) == ENABLED
                && li.mOnTouchListener.onTouch(this, event)) {
            result = true;
        }

        if (!result && onTouchEvent(event)) {
            result = true;
        }
        return result;
    }

```

该方法里主要逻辑为判断是否有onTouchListener，如果有，并且其onTouch方法返回true，则之后所有的事件都由onTouch方法处理；如果没有或者onTouchListener的onTouch方法返回false，则执行onTouchEvent方法；

首先触发的事件为ACTION_DOWN，这时候会给View标记为prepressed（预点击），延时100ms，如果还没有执行其它事件，这时候就标记为pressed（点击），记录为预长按，延时400ms，如果也没有执行其它事件，则触发onLongClick事件，当然前提是onLongClickListener不为空；源码如下：

```
	@Override
	public boolean onTouchEvent(MotionEvent event) {
        //...其它代码
        if(((viewFlags & CLICKABLE) == CLICKABLE ||
                (viewFlags & LONG_CLICKABLE) == LONG_CLICKABLE) ||
                (viewFlags & CONTEXT_CLICKABLE) == CONTEXT_CLICKABLE){
			switch (event.getAction()) {
				case MotionEvent.ACTION_DOWN:
					mPrivateFlags |= PFLAG_PREPRESSED;
					if (mPendingCheckForTap == null) {
						mPendingCheckForTap = new CheckForTap();
					}
					mPendingCheckForTap.x = event.getX();
					mPendingCheckForTap.y = event.getY();
					postDelayed(mPendingCheckForTap,
							ViewConfiguration.getTapTimeout());
				break;
        	}  
        	return true;
        }
    	return false;
    }
    
    private final class CheckForTap implements Runnable {
        @Override
        public void run() {
            mPrivateFlags &= ~PFLAG_PREPRESSED;
            setPressed(true, x, y);
            checkForLongClick(ViewConfiguration.getTapTimeout(), x, y);
        }
    }
    
    private void checkForLongClick(int delayOffset, float x, float y) {
        if ((mViewFlags & LONG_CLICKABLE) == LONG_CLICKABLE) {
            mHasPerformedLongPress = false;

            if (mPendingCheckForLongPress == null) {
                mPendingCheckForLongPress = new CheckForLongPress();
            }
            postDelayed(mPendingCheckForLongPress,
                    ViewConfiguration.getLongPressTimeout() - delayOffset);
        }
    }
    
    private final class CheckForLongPress implements Runnable {
        @Override
        public void run() {
            if (isPressed() && (mParent != null)
                    && mOriginalWindowAttachCount == mWindowAttachCount) {
                if (performLongClick(mX, mY)) {
                    mHasPerformedLongPress = true;
                }
            }
        }
    }
    
    private boolean performLongClickInternal(float x, float y) {
        boolean handled = false;
        final ListenerInfo li = mListenerInfo;
        if (li != null && li.mOnLongClickListener != null) {
            handled = li.mOnLongClickListener.onLongClick(View.this);
        }
        return handled;
    }
```

其中performLongClick方法就是得到长按的返回值，改变mHasPerformedLongPress的值，表示长按是否返回true；

回到主路线，下边就是执行ACTION_MOVE，源码如下：

```
	case MotionEvent.ACTION_MOVE:
		if (!pointInView(x, y, mTouchSlop)) {
			removeTapCallback();
			if ((mPrivateFlags & PFLAG_PRESSED) != 0) {
				removeLongPressCallback();
				setPressed(false);
			}
		}
	break;
	
	public boolean pointInView(float localX, float localY, float slop) {
        return localX >= -slop && localY >= -slop && localX < ((mRight - mLeft) + slop) &&
                localY < ((mBottom - mTop) + slop);
    }
```

代码很简单，就是移动的距离超过最小滑动距离后，就要移除掉pressed标记，和长按标记；

然后执行的就是ACTION_UP，源码如下：

```
	case MotionEvent.ACTION_UP:
	    if (!mHasPerformedLongPress && !mIgnoreNextUpEvent) {
        removeLongPressCallback();
        if (!focusTaken) {
            if (mPerformClick == null) {
                mPerformClick = new PerformClick();
            }
            if (!post(mPerformClick)) {
                performClick();
            }
        }
    }
	break;
	
	public boolean performClick() {
        final boolean result;
        final ListenerInfo li = mListenerInfo;
        if (li != null && li.mOnClickListener != null) {
            playSoundEffect(SoundEffectConstants.CLICK);
            li.mOnClickListener.onClick(this);
            result = true;
        } else {
            result = false;
        }

        sendAccessibilityEvent(AccessibilityEvent.TYPE_VIEW_CLICKED);
        return result;
    }

```

这里也很简单就是表示长按没有触发，或者出发后长按返回false，则就去执行onClickListener的onClick方法，当然前提是onClickListener不为空；

onTouchEvent当然还有其它的事件，例如ACTION_CANCEL（触摸的位置超出View范围）、ACTION_POINTER_DOWN、ACTION_POINTER_UP等等，具体含义就不做介绍。

### ViewGroup的事件体系

对于View来说，不同的有两点，一是dispatchTouchEvent方法有所不同，二是多了一个onInterceptTouchEvent方法；而onTouchEvent的流程却是一样的；

首先来看dispatchTouchEvent，该类比较复杂，只抽取重要的部分，代码如下：

```
	@Override
	public boolean dispatchTouchEvent(MotionEvent ev) {
		//...其他代码
		final boolean intercepted;
		if (actionMasked == MotionEvent.ACTION_DOWN
                    || mFirstTouchTarget != null) {
			final boolean disallowIntercept = (mGroupFlags & FLAG_DISALLOW_INTERCEPT) != 0;
			if (!disallowIntercept) {
				intercepted = onInterceptTouchEvent(ev);
				ev.setAction(action); 
			} else {
				intercepted = false;
			}
		} else {
			intercepted = true;
		}
		
		//...
		
		if (intercepted || mFirstTouchTarget != null) {
			ev.setTargetAccessibilityFocus(false);
		}	
		
		//...
		
		if (!canceled && !intercepted) {
			if (actionMasked == MotionEvent.ACTION_DOWN
                        || (split && actionMasked == MotionEvent.ACTION_POINTER_DOWN)
                        || actionMasked == MotionEvent.ACTION_HOVER_MOVE) {
                        
				final int childrenCount = mChildrenCount;
				if (newTouchTarget == null && childrenCount != 0) {
					final View[] children = mChildren;
					
					for (int i = childrenCount - 1; i >= 0; i--) {
					
						final int childIndex = getAndVerifyPreorderedIndex(
                                    childrenCount, i, customOrder);
                                    
						final View child = getAndVerifyPreorderedView(
                                    preorderedList, children, childIndex);
                                    
						if (dispatchTransformedTouchEvent(ev, false, child, idBitsToAssign)) {
							 //...其它代码
							newTouchTarget = addTouchTarget(child, idBitsToAssign);
							alreadyDispatchedToNewTouchTarget = true;
							break;
						}
					｝
				｝
			｝
		｝
		if (mFirstTouchTarget == null) {
			handled = dispatchTransformedTouchEvent(ev, canceled, null,
                        TouchTarget.ALL_POINTER_IDS);
		} else {
			//...
			dispatchTransformedTouchEvent(ev, cancelChild,
                                target.child, target.pointerIdBits);
			//...
                                
		｝
	}
	
	private static View getAndVerifyPreorderedView(ArrayList<View> preorderedList, View[] children,
            int childIndex) {
        final View child;
        if (preorderedList != null) {
            child = preorderedList.get(childIndex);
            if (child == null) {
                throw new RuntimeException("Invalid preorderedList contained null child at index "
                        + childIndex);
            }
        } else {
            child = children[childIndex];
        }
        return child;
    }
    
    private boolean dispatchTransformedTouchEvent(MotionEvent event, boolean cancel,
            View child, int desiredPointerIdBits) {
        final boolean handled;

        final int oldAction = event.getAction();
        if (cancel || oldAction == MotionEvent.ACTION_CANCEL) {
            event.setAction(MotionEvent.ACTION_CANCEL);
            if (child == null) {
                handled = super.dispatchTouchEvent(event);
            } else {
                handled = child.dispatchTouchEvent(event);
            }
            event.setAction(oldAction);
            return handled;
        }

		//...其它代码

        if (child == null) {
            handled = super.dispatchTouchEvent(transformedEvent);
        } else {
        	//...其它代码
            handled = child.dispatchTouchEvent(transformedEvent);
        }

        transformedEvent.recycle();
        return handled;
    }
	
```

代码比较多，但是意思不难，用伪代码表达的话就很容易明白了，如下：

```
public boolean dispatchTouchEvent(MotionEvent ev){
	if(onInterceptTouchEvent(ev)){
		return onTouchEvent(ev);
	}else{
		return child.dispatchTouchEvent(ev);
	}
}
```
当然伪代码表达的肯定是不完全的，接下来，稍微详细的说一下我的理解，首先判断的还是是否会被拦截，这里有个标记，FLAG_DISALLOW_INTERCEPT，子View通过给其父容器设置这个标记就能让父容器的onInterceptTouchEvent方法无效，方法如下：

```
	@Override
    public void requestDisallowInterceptTouchEvent(boolean disallowIntercept) {

        if (disallowIntercept == ((mGroupFlags & FLAG_DISALLOW_INTERCEPT) != 0)) {
            // We're already in this state, assume our ancestors are too
            return;
        }

        if (disallowIntercept) {
            mGroupFlags |= FLAG_DISALLOW_INTERCEPT;
        } else {
            mGroupFlags &= ~FLAG_DISALLOW_INTERCEPT;
        }

        // Pass it up to our parent
        if (mParent != null) {
            mParent.requestDisallowInterceptTouchEvent(disallowIntercept);
        }
    }
```
让该子View的父容器都打上该标记，就能直接跳过拦截了，这个在解决滑动冲突时，是比较重要的方法。

回到主路线，如果没有该标记，则通过onInterceptTouchEvent方法判断是否拦截，如果拦截，则确定了接受事件的targetView，之后就会把所有的事件都给targetView，当然所有父类的分发和拦截方法也会依次调用；如果未拦截，就要依次循环子View的dispatchTouchEvent方法，判断是否消费了事件，具体的过程就是View的事件体系中所描述的，消费了事件，则也能确定targetView，同样就会把所有的事件都给targetView；如果子View都未消费，最后就会传回到Activity的onTouchEvent事件中处理。

最后我们再来看一下onInterceptTouchEvent，代码比较简单，如下：

```
	public boolean onInterceptTouchEvent(MotionEvent ev) {
        if (ev.isFromSource(InputDevice.SOURCE_MOUSE)
                && ev.getAction() == MotionEvent.ACTION_DOWN
                && ev.isButtonPressed(MotionEvent.BUTTON_PRIMARY)
                && isOnScrollbarThumb(ev.getX(), ev.getY())) {
            return true;
        }
        return false;
    }
```

那么ViewGroup的事件体系大体上就走完了，之后再上一张ViewGroup的事件流程图，如下：

![](../images/ViewGroup事件体系.png)

	



