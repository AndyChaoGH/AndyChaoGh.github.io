---
layout: post
title: Android学习感悟之View之一绘制流程
author: arvinljw
---

上一篇介绍了Activity，过了这么久，没有再开始写下一篇，一是工作加班多没心情写，而是确实也有点懒，今天终于开始了第二篇的感悟——View，这里会分三个部分，绘制，事件，解决滑动冲突。

# 简介

说到View，只要是接触过Android的朋友，应该都知道是什么东西，他就是承载各种数据显示的控件，比如文本常常使用TextView来显示，图片－>ImageView，列表－>RecyclerView；在这篇文章中会从以下几个方面来介绍我所感悟的View。

本文包括：

* 如何自定义View；

## 自定义View

首先我们需要了解为什么要自定义View？在我看来，比较常见的其目的有以下两种：

* 解决系统提供的View难以满足界面实现的需求；
* 封装某一特定View组合，实现代码的复用；

那要如何实现自定义View，下边会用一些简单的例子来说明；当然实现方式也是多种多样，我认为可以分为以下几种情况：

* 自定义View：
	
	* 继承View并进一步实现；
	* 继承已有的View并进一步实现。

* 自定义ViewGroup：

	* 继承ViewGroup并进一步实现；
	* 继承已有的ViewGroup并实现；

接下来借用例子逐一实现，首先从自定义View开始，继承View并实现，接下来要实现的是以左上角为直角的等边直角三角形；

**第一步：创建一个View**取名叫TriangleView继承自View并实现其构造方法，如下：

```
public class TriangleView extends View {
	public TriangleView(Context context) {
        this(context, null);
    }

    public TriangleView(Context context, AttributeSet attrs) {
        this(context, attrs, 0);
    }

    public TriangleView(Context context, AttributeSet attrs, int defStyleAttr) {
        super(context, attrs, defStyleAttr);
    }
}
```

这一步中有一个细节和一个要点。

细节就是构造函数之间的调用，如果在不清楚你所继承的View构造方法的调用中是否有其他配置的话（例如主题样式），最好还是直接调用对应参数的super构造函数；

要点就是必须重写带有Attibuteset参数的构造参数，这样我们才能拿到在布局的xml文件中的参数。

**第二步：想好自定义View所包含的属性**，这里我们只需要三角形的填充色，实现如下：

首先在资源文件中创建一个declare-styleable资源，一般写在values下的attrs文件中，如下：

```
	<declare-styleable name="TriangleView">
        <attr name="fillColor" format="color" />
    </declare-styleable>
```

这里需要注意的是名字最好与自定义View的名字一致，方便查找，也更加规范；

attr标签中就是自定义的属性，包含了名称和类型，其中类型有很多种，下边也整理了一个表格，描述各个属性的含义，如下：

|format|meaning|
|:------:|:-------|
|color   |颜色值，如："#ff0000"或"R.color.colorAccent"|
|dimension|尺寸，如：单位为sp,dp，px的值|
|reference|资源id，如："R.drawable.img_loading"|
|boolean|布尔类型，如：true or false|
|float|浮点小数，如：0.1|
|integer|整数，如：10|
|string|字符串，如："你好啊"或"R.string.app_name"|
|fraction|百分比，如：20%|
|enum|枚举，具体使用请参照LinearLayout的orientation属性|
|flag|标记，即某个属性可以有多个标记，相对于枚举而言，枚举只能选一个值，这个可以选多个，例如layout_gravity属性|

定义好了，接下来就是设置属性和获取属性了，属性的设置可在xml布局文件中设置，例如：

```
<com.arvin.example.allibraryexample.TriangleView
	xmlns:app="http://schemas.android.com/apk/res-auto"
	android:layout_width="wrap_content"
	android:layout_height="wrap_content"
	app:fillColor="@color/colorAccent" />
```

注意，在使用自定义属性是需要该View或者其父View中加上

**xmlns:app="http://schemas.android.com/apk/res-auto"**

其作用就是能自动找到该View所定义的属性，使用方式就是

**app:自定义属性**

然后就是获取属性，如下：

```
	super(context, attrs, defStyleAttr);
	TypedArray typedArray = context.obtainStyledAttributes(attrs, 	R.styleable.TriangleView);
	fillColor = typedArray.getColor(R.styleable.TriangleView_fillColor, 	Color.RED);
	typedArray.recycle();
```

attrs中包含了，在xml布局文件中所有的属性，系统自带属性可通过调用父类的构造函数获得，自定义属性，即可通过TypedArray获得，每种自定义属性的类型都在TypedArray中对应着一个get方法，等我们取完搜索属性后，记得要recycler掉，原因就是为了复用这个TypedArray，具体原因在这里就不再细说了，总之这个模式这样用很好。

**第三步，定义View的大小**

这其中就涉及到一个比较重要的方法了，需要重写

```
	@Override
	protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec){
	}
```

方法，然后设置宽度和高度，这里有一点小常识，Android中所有的View外形都是一个矩形，只是绘制时，不同的方法导致形状不同而已。言归正传，这时候就需要讲解一下计算大小的规则了，Android提供了一个MeasureSpec，包含了测量模式，以及测量尺寸，测量模式包括：

* UNSPECIFIED

	父视图不对子视图有任何约束，它可以达到所期望的任意尺寸。比如 ListView、ScrollView，一般自定义 View 中用不到
	
* EXACTLY

	父视图为子视图指定一个确切的尺寸，而且无论子视图期望多大，它都必须在该指定大小的边界内，对应的属性为 match_parent 或具体值，比如 100dp，父控件可以通过MeasureSpec.getSize(measureSpec)直接得到子控件的尺寸。

* AT_MOST

	父视图为子视图指定一个最大尺寸。子视图必须确保它自己所有子视图可以适应在该尺寸范围内，对应的属性为 wrap_content，这种模式下，父控件无法确定子 View 的尺寸，只能由子控件自己根据需求去计算自己的尺寸，这种模式就是我们自定义视图需要实现测量逻辑的情况。
	
简单的了解过后我们参考onMeasure的默认实现，如下：

```
	protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
		setMeasuredDimension(getDefaultSize(getSuggestedMinimumWidth(), widthMeasureSpec),
                getDefaultSize(getSuggestedMinimumHeight(), heightMeasureSpec));
	}

	protected int getSuggestedMinimumWidth() {
	return (mBackground == null) ? mMinWidth : max(mMinWidth, mBackground.getMinimumWidth());
}

	public int getMinimumHeight() {
 	  return mMinHeight;
	}

	public static int getDefaultSize(int size, int measureSpec) {
		int result = size;
		int specMode = MeasureSpec.getMode(measureSpec);
		int specSize = MeasureSpec.getSize(measureSpec);

   		switch (specMode) {
			 case MeasureSpec.UNSPECIFIED:
			 	result = size;
  	    		break;
			case MeasureSpec.AT_MOST:
			case MeasureSpec.EXACTLY:
 	     		result = specSize;
	      		break;
 	  	}
 	  return result;
	}
```

代码很简单，可以发现，其实就是通过调用setMeasuredDimension方法，设置了宽度和高度，默认情况下，wrap_content时，size为0，而我希望有一点高度和宽度，所以，我们自己写一个getSize方法，并让测量模式为wrap_content时，有一个默认的大小，具体如下：

```
	private int getSize(int size, int measureSpec) {
		int result = size;
		int specMode = MeasureSpec.getMode(measureSpec);
		int specSize = MeasureSpec.getSize(measureSpec);

		switch (specMode) {
			case MeasureSpec.UNSPECIFIED:
				result = size;
				break;
			case MeasureSpec.AT_MOST:
				result = dp2px(DEFAULT_SIZE);
				break;
			case MeasureSpec.EXACTLY:
				result = specSize;
				break;
		}
	return result;
	}

	private int dp2px(float dp) {
		return (int) (dp * getResources().getDisplayMetrics().density + 0.5f);
	}
```

然后画出来的三角形，我希望是等边直角三角形，所以要让宽和高一样大，这样简单处理一下后，大小就设置完了，具体的onMeasure实现如下：

```
	@SuppressWarnings("SuspiciousNameCombination")
	@Override
	protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
		int width = getSize(getSuggestedMinimumWidth(), widthMeasureSpec);
		int height = getSize(getSuggestedMinimumWidth(), heightMeasureSpec);

		if (width > height) {
		width = height;
		} else {
			height = width;
		}

		setMeasuredDimension(width, height);
	}
```

**第四步，重写onDraw方法**，这里就是绘制三角形，代码很简单，就不解释了，如下：

```
	//在构造函数中调用，初始化画笔
	private void init() {
	mPaint = new Paint();
		mPaint.setAntiAlias(true);
		mPaint.setColor(fillColor);
	}

	@SuppressLint("DrawAllocation")
	@Override
	protected void onDraw(Canvas canvas) {
		super.onDraw(canvas);

		Path path = new Path();
		path.moveTo(0, 0);
		path.lineTo(getWidth(), 0);
		path.lineTo(0, getHeight());
		path.close();

		canvas.drawPath(path, mPaint);
	}
```

这样，以左上角为直角的等边直角三角形就画好了。

对于继承特定View来实现自定义View，也不外乎上边四步，所以就不再介绍了；

接下来就是实现一个自定义ViewGroup，其中属性的创建和获取以及绘制与自定义View一致，就不再赘述，而更为重要的就是onMeasure以及onLayout方法，ViewGroup在我看来更像一个管理者，把自己的子View管理起来，管理其大小，安排其位置。

下面，我们就来做一个自定义的ViewGroup，功能就是流布局，从上到下，从左到右，把子View依次排列；我们可以想象一下实现思路，首先，其ViewGroup的大小如何设置，很明显，需要分别考虑AT_MOST和EXACTLY两种模式(UNSPECIFIED这里不作考虑)下宽度和高度如何计量，思路如下：

* AT_MOST

	宽度：只要等于宽度最大的一行的宽度加上paddingLeft和paddingRight后与父容器能给的最大块度比较，较小的就是最终的宽度；
	
	高度：把每一行的高度加起来再加上paddingTop和paddingBottom后与父容器能给的最大高度比较，较小的就是最终的高度；
	
* EXACTLY
	
	宽度：其父容器能给的最大宽度；
	
	高度：其父容器能给的最大高度；
	
其中有一个逻辑就是什么时候换行，其实很明显，就是当这一行累积的子View的宽度大于容器的宽度时，最后一个累积的View需要换行。换行后的高度，就在上一行的下边开始。

下边就是具体的代码：

```
	@Override
    protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
        int widthMode = MeasureSpec.getMode(widthMeasureSpec);
        int widthSize = MeasureSpec.getSize(widthMeasureSpec);
        int heightMode = MeasureSpec.getMode(heightMeasureSpec);
        int heightSize = MeasureSpec.getSize(heightMeasureSpec);

        int horizontalPadding = getPaddingLeft() + getPaddingRight();
        int verticalPadding = getPaddingTop() + getPaddingBottom();

        int width = 0;
        int height = 0;

        int lineWidth = 0;
        int lineHeight = 0;

        for (int i = 0; i < getChildCount(); i++) {
            View child = getChildAt(i);
            MarginLayoutParams params = (MarginLayoutParams) child.getLayoutParams();

            measureChild(child, widthMeasureSpec, heightMeasureSpec);

            int childWidth = child.getMeasuredWidth() + params.leftMargin + params.rightMargin;
            int childHeight = child.getMeasuredHeight() + params.topMargin + params.bottomMargin;

            if (childWidth + lineWidth > widthSize - horizontalPadding) {
                width = Math.max(width, lineWidth);
                height += lineHeight;

                lineWidth = childWidth;
                lineHeight = childHeight;
            } else {
                lineWidth += childWidth;
                lineHeight = Math.max(childHeight, lineHeight);
            }

            if (i == getChildCount() - 1) {
                width = Math.max(width, lineWidth) + horizontalPadding;
                height += lineHeight + verticalPadding;
            }
        }

        setMeasuredDimension(widthMode == MeasureSpec.EXACTLY ? widthSize : width,
                heightMode == MeasureSpec.EXACTLY ? heightSize : height);
    }
```

首先获取到父容器为该view分配的最大测量尺寸以及测量模型，目的就是限制最大的宽度，这里高度就没有做限制，子View有多少都给显示出来；

定义变量，width和height表示最大的宽度和高度；lineWidth和lineHeight为了记录当前行最大宽度和高度；horizontalPadding和verticalPadding就是pading，很简单就不解释了。

然后就是遍历所有子View，通过measureChild方法，测量一下子View，然后就能获取到子View的大小了（这里我们还支持了margin）；

接下来就是最重要的逻辑了，就是是否换行的逻辑，即当加上当前子View后宽度是否比内容最大宽度大，大则换行，小则继续累加；换行后，行宽从新计算，行高累积，遍历完后，就能得到一个承载搜索子View的宽高，然后再看测量模型，来决定如何设值，MeasureSpec.EXACTLY则就是父容器所设定的值，MeasureSpec.AT_MOST则就是计算出来的宽高，这样就把这个子View的大小测量完毕了。

其中又一点需要注意，是让View具有margin，则需要生成一个LayoutParams，如下：

```
    @Override
    public LayoutParams generateLayoutParams(AttributeSet attrs) {
        return new MarginLayoutParams(getContext(), attrs);
    }
```

最后就是把子View排位，这一步和测量时逻辑几乎相同，唯一不同就是，需要让子View调用layout方法，安排自己的位置，代码如下：

```
    @Override
    protected void onLayout(boolean changed, int l, int t, int r, int b) {
        int left = getPaddingLeft();
        int top = getPaddingTop();

        int horizontalPadding = getPaddingLeft() + getPaddingRight();

        int lineWidth = 0;
        int lineHeight = 0;

        for (int i = 0; i < getChildCount(); i++) {
            View child = getChildAt(i);
            if (child.getVisibility() == View.GONE) {
                continue;
            }
            MarginLayoutParams params = (MarginLayoutParams) child.getLayoutParams();

            int childWidth = child.getMeasuredWidth() + params.leftMargin + params.rightMargin;
            int childHeight = child.getMeasuredHeight() + params.topMargin + params.bottomMargin;

            int cl, ct, cr, cb;

            if (childWidth + lineWidth > getWidth() - horizontalPadding) {

                left = getPaddingLeft();
                top += lineHeight;

                cl = left + params.leftMargin;
                ct = top + params.topMargin;
                cr = cl + child.getMeasuredWidth();
                cb = ct + child.getMeasuredHeight();

                left += childWidth;

                lineWidth = childWidth;
                lineHeight = childHeight;
            } else {
                cl = left + params.leftMargin;
                ct = top + params.topMargin;
                cr = cl + child.getMeasuredWidth();
                cb = ct + child.getMeasuredHeight();

                left += childWidth;

                lineWidth += childWidth;
                lineHeight = Math.max(childHeight, lineHeight);
            }

            child.layout(cl, ct, cr, cb);
        }
    }
```

变量解释：
left就是每一个View的最左边，top就是每一个View的最上边；cl,ct,cr,cb分别表示当前子View的左上右下的位置；

逻辑结构都是一样，就是在每一个View判断完是否换行后，计算出其坐标，并layout，计算过程也很简单，如果不换行就改变left的值，保持top不变，以及根据子View设置右下的位置，换行时，初始化left，top值在上一行的下边，这一步layout后记得更改left的值，这样子View的布局就搞定了。测试运行，发现刚刚好。

还有其他的一些自定义ViewGroup，但是掌握了最核心的原理后，发现每一个都是这样，所以还有一些实现方式就不再距离说明了。

这样自定义View就完成了。
	



