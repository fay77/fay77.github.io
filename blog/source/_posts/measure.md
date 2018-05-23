---
title: 自定义View之OnMeasure过程浅析
date: 2018-05-06 13:32:54
tags:
---
<!--more-->


----------
>自定义`view`的流程分为`measure`、`layout`、`draw`三个主要步骤，今天我们通过源码来分下下measure的过程我们从顶级`view`开始，顶级`view`即`DecorView`，`view`的事件都是先经过这个`DecorView`,接下来我们来看看这个`DecorView`的`MeasureSpe`c的创建过程： 
进入`ViewRootImpl`中，查看`measureHierarchy`方法，有如下代码：


```
final DisplayMetrics packageMetrics = res.getDisplayMetrics();  
           res.getValue(com.android.internal.R.dimen.config_prefDialogWidth, mTmpValue, true);  
           int baseSize = 0;  
           if (mTmpValue.type == TypedValue.TYPE_DIMENSION) {  
               baseSize = (int)mTmpValue.getDimension(packageMetrics);  
           }  
           if (DEBUG_DIALOG) Log.v(mTag, "Window " + mView + ": baseSize=" + baseSize  
                   + ", desiredWindowWidth=" + desiredWindowWidth);  
           if (baseSize != 0 && desiredWindowWidth > baseSize) {  
               childWidthMeasureSpec = getRootMeasureSpec(baseSize, lp.width);  
               childHeightMeasureSpec = getRootMeasureSpec(desiredWindowHeight, lp.height);  
               performMeasure(childWidthMeasureSpec, childHeightMeasureSpec);  
```

这里只是截选一部分的源码， 我们看到这个`baseSize`， 其实就是屏幕的尺寸大小， 获取宽的`MeasureSpc`的方法：    

    childWidthMeasureSpec = getRootMeasureSpec(baseSize, lp.width);

这里传入的参数是屏幕尺寸以及DecorView自身的大小， 接着我们来看 `getRootMeasureSpec`方法：



```
private static int getRootMeasureSpec(int windowSize, int rootDimension) {  
       int measureSpec;  
       switch (rootDimension) {  
  
       case ViewGroup.LayoutParams.MATCH_PARENT:  
           // Window can't resize. Force root view to be windowSize.  
           measureSpec = MeasureSpec.makeMeasureSpec(windowSize, MeasureSpec.EXACTLY);  
           break;  
       case ViewGroup.LayoutParams.WRAP_CONTENT:  
           // Window can resize. Set max size for root view.  
           measureSpec = MeasureSpec.makeMeasureSpec(windowSize, MeasureSpec.AT_MOST);  
           break;  
       default:  
           // Window wants to be an exact size. Force root view to be that size.  
           measureSpec = MeasureSpec.makeMeasureSpec(rootDimension, MeasureSpec.EXACTLY);  
           break;  
       }  
       return measureSpec;  
   }  
```

就是这个方法确定了`DecorView`的`MeasureSpec`， 这里分了三种情况，

>  1. 如果传入的view大小为`math_parent`，那么这个view的mode为`EXACTLY`， 大小为屏幕的尺寸.
 2. 如果传入的view大小为`wrap_content`，那么这个view的mode为`AT_MOST`,大小为屏幕的尺寸.
 3. 如果传入的view大小为一个具体的值，那么这个view的mode为`EXACTLY`，大小为view本身大小。

以上就是DecorView的`MeaureSpec`的整个创建的过程了。 

**看了顶级view之后我们来看普通的view， 普通的view的`measure`过程是由`viewgroup`传递过来的，接着我们来看看`viewgroup`的`measureChildWithMargins`方法：**


```
protected void measureChildWithMargins(View child,  
           int parentWidthMeasureSpec, int widthUsed,  
           int parentHeightMeasureSpec, int heightUsed) {  
       final MarginLayoutParams lp = (MarginLayoutParams) child.getLayoutParams();  
  
       final int childWidthMeasureSpec = getChildMeasureSpec(parentWidthMeasureSpec,  
               mPaddingLeft + mPaddingRight + lp.leftMargin + lp.rightMargin  
                       + widthUsed, lp.width);  
       final int childHeightMeasureSpec = getChildMeasureSpec(parentHeightMeasureSpec,  
               mPaddingTop + mPaddingBottom + lp.topMargin + lp.bottomMargin  
                       + heightUsed, lp.height);  
  
       child.measure(childWidthMeasureSpec, childHeightMeasureSpec);  
   }  
```

**这个方法获得了子view的`MeasureSpec`，并且将其传入子view的`measure`方法中， 这里重点来看下`viewgroup`是如何创建子view的`MeasuerSpec`的。来看`getChildMeasureSpec`方法内部的实现：**

```
public static int getChildMeasureSpec(int spec, int padding, int childDimension) {  
      int specMode = MeasureSpec.getMode(spec);  
      int specSize = MeasureSpec.getSize(spec);  
  
      int size = Math.max(0, specSize - padding);  
  
      int resultSize = 0;  
      int resultMode = 0;  
  
      switch (specMode) {  
      // Parent has imposed an exact size on us  
      case MeasureSpec.EXACTLY:  
          if (childDimension >= 0) {  
              resultSize = childDimension;  
              resultMode = MeasureSpec.EXACTLY;  
          } else if (childDimension == LayoutParams.MATCH_PARENT) {  
              // Child wants to be our size. So be it.  
              resultSize = size;  
              resultMode = MeasureSpec.EXACTLY;  
          } else if (childDimension == LayoutParams.WRAP_CONTENT) {  
              // Child wants to determine its own size. It can't be  
              // bigger than us.  
              resultSize = size;  
              resultMode = MeasureSpec.AT_MOST;  
          }  
          break;  
  
      // Parent has imposed a maximum size on us  
      case MeasureSpec.AT_MOST:  
          if (childDimension >= 0) {  
              // Child wants a specific size... so be it  
              resultSize = childDimension;  
              resultMode = MeasureSpec.EXACTLY;  
          } else if (childDimension == LayoutParams.MATCH_PARENT) {  
              // Child wants to be our size, but our size is not fixed.  
              // Constrain child to not be bigger than us.  
              resultSize = size;  
              resultMode = MeasureSpec.AT_MOST;  
          } else if (childDimension == LayoutParams.WRAP_CONTENT) {  
              // Child wants to determine its own size. It can't be  
              // bigger than us.  
              resultSize = size;  
              resultMode = MeasureSpec.AT_MOST;  
          }  
          break;  
  
      // Parent asked to see how big we want to be  
      case MeasureSpec.UNSPECIFIED:  
          if (childDimension >= 0) {  
              // Child wants a specific size... let him have it  
              resultSize = childDimension;  
              resultMode = MeasureSpec.EXACTLY;  
          } else if (childDimension == LayoutParams.MATCH_PARENT) {  
              // Child wants to be our size... find out how big it should  
              // be  
              resultSize = View.sUseZeroUnspecifiedMeasureSpec ? 0 : size;  
              resultMode = MeasureSpec.UNSPECIFIED;  
          } else if (childDimension == LayoutParams.WRAP_CONTENT) {  
              // Child wants to determine its own size.... find out how  
              // big it should be  
              resultSize = View.sUseZeroUnspecifiedMeasureSpec ? 0 : size;  
              resultMode = MeasureSpec.UNSPECIFIED;  
          }  
          break;  
      }  
      //noinspection ResourceType  
      return MeasureSpec.makeMeasureSpec(resultSize, resultMode);  
  }  
```
这个方法很长，但是我们只需要注意到`AT_MOST`跟`EXACTLY`这两种情况就行，稍微分析下这个过程：
首先要理解这个方法的三个参数， 第一个是父view的`MeasureSpec`, 第二个是父view已占用的大小，第三个是view的`LayoutParams`的大小，如果不理解可以看看`ViewGroup`的`MeasureChildWithMargins`方法中的调用：


```
final int childWidthMeasureSpec = getChildMeasureSpec(parentWidthMeasureSpec,  
               mPaddingLeft + mPaddingRight + lp.leftMargin + lp.rightMargin  
                       + widthUsed, lp.width);  

```
>第二个参数很长 ， `mPaddingLeft` + `mPaddingRight` + `lp.leftMargin` + `lp.rightMargin` + `withUsed`  ，这些所有的值都有一个共同特点，就是这些位置是不能摆放任何view的，即父view已经占用的地盘，现在是不是对参数更加理解了呢。 


接着我们回到`getChildMeasureSpec`方法中继续看看`viewGroup`到底是怎么创建view的`MeasureSpec`的。


第一步： 根据参数一，即传入的父view的`MeasureSpec`获得父view的`Mode`和`Size`。这里的第三行代码：


    int size = Math.max(0, specSize - padding);  

这个`size`表示取0与父容器中可占用的位置的最大值，可以直接理解为父view的大小。

第二步：根据父view的Mode分情况处理， 到这一步我们应该就清楚为什么说view的大小是由父view的`MeasureSpec`与本身`LayoutParmas`大小共同决定的吧。

这里我们依然只看`AT_MOST`跟`EXACTLY`两种情况， 


```
switch (specMode) {  
      // Parent has imposed an exact size on us  
      case MeasureSpec.EXACTLY:  
          if (childDimension >= 0) {  
              resultSize = childDimension;  
              resultMode = MeasureSpec.EXACTLY;  
          } else if (childDimension == LayoutParams.MATCH_PARENT) {  
              // Child wants to be our size. So be it.  
              resultSize = size;  
              resultMode = MeasureSpec.EXACTLY;  
          } else if (childDimension == LayoutParams.WRAP_CONTENT) {  
              // Child wants to determine its own size. It can't be  
              // bigger than us.  
              resultSize = size;  
              resultMode = MeasureSpec.AT_MOST;  
          }  
          break;  
  
      // Parent has imposed a maximum size on us  
      case MeasureSpec.AT_MOST:  
          if (childDimension >= 0) {  
              // Child wants a specific size... so be it  
              resultSize = childDimension;  
              resultMode = MeasureSpec.EXACTLY;  
          } else if (childDimension == LayoutParams.MATCH_PARENT) {  
              // Child wants to be our size, but our size is not fixed.  
              // Constrain child to not be bigger than us.  
              resultSize = size;  
              resultMode = MeasureSpec.AT_MOST;  
          } else if (childDimension == LayoutParams.WRAP_CONTENT) {  
              // Child wants to determine its own size. It can't be  
              // bigger than us.  
              resultSize = size;  
              resultMode = MeasureSpec.AT_MOST;  
          }  
          break;  
```

这里有个比较难以理解的值就是`childDimension` > 0 ，这个其实就表示`view`的大小是一个具体的值比如100dp , 因为view的`match_parent`和 `wrap_content`在系统内部定义的都是负数，一个是-1. 一个是-2  ，所以判断`childDimension` > 0即，view的大小为一个具体的值。 
接着就比较好理解了， 我们来稍微总结下：

>无论父view是`match_parent`还是 `wrap_content` ，只要view是一个具体的值，view的Mode永远都是`EXACTLY`, 大小均是view本身定义的大小。

>父view模式如果是`EXACTLY`,  --->  子view如果是`mathch_parent` ，那么子view的大小是父view的大小，模式也跟父view一样为`EXACTLY`.  子view如果是`wrap_content`， 大小还是父view的大小， 模式为`AT_MOST`

>父view模式如果是`AT_MOST` ,  --- >  子view如果是`math_parent`， 那么子view大小为父view大小， 模式与父view一样都是`AT_MOST`,  子view如果是`wrap_content`， 子view大小为父view大小， 模式为`AT_MOST`


----------


**上面说的有点绕，但其实我们只需要记住一点， 无论上面那种情况，子view在`wrap_content`下，大小都是父view的大小，  到这里我们是不是就能理解为什么在自定义view的过程中如果不重写`onMeasure`， `wrap_content`是和`match_parent`是一个效果了吧。**



> 以上过程是viewGroup中创建子view的`MeasureSpec`的过程，
> 有了这个MeasureSpec，测量子view大小就很简单了，我们可以看到在ViewGroup获取到子view的`MeasureSpec`之后，传入到子view的`measure`方法中：


    child.measure(childWidthMeasureSpec, childHeightMeasureSpec);  


----------


**进入到view的`measure`方法,**


不知不觉我们已经从`viewgroup`进入到了`view`的测量过程，



这里是不是突然意识到，`ViewGroup`根本没有测绘自己本身啊， 只是获取到子view的`MeasureSpec`然后传入子view的`measure`方法里去，这是因为`ViewGroup`是个抽象类，本身并没有定义测量的过程， `ViewGroup`的`onMeasure`需要各个子类去实现，比如`LinearLayout` 、 `RelativeLayout`等等，并且每个子类的测量过程都不一样，这个我们后面会讲， 现在我们还是接着看view的`Measure`过程。



上面说到`viewgroup`将创建的子view的`MeasureSpec`传入到了view的`Measure`方法中， 那么我们就来看看View的`Measure`方法：



```
public final void measure(int widthMeasureSpec, int heightMeasureSpec) {  
       boolean optical = isLayoutModeOptical(this);  
       if (optical != isLayoutModeOptical(mParent)) {  
           Insets insets = getOpticalInsets();  
           int oWidth  = insets.left + insets.right;  
           int oHeight = insets.top  + insets.bottom;  
           widthMeasureSpec  = MeasureSpec.adjust(widthMeasureSpec,  optical ? -oWidth  : oWidth);  
           heightMeasureSpec = MeasureSpec.adjust(heightMeasureSpec, optical ? -oHeight : oHeight);  
       }  
  
       // Suppress sign extension for the low bytes  
       long key = (long) widthMeasureSpec << 32 | (long) heightMeasureSpec & 0xffffffffL;  
       if (mMeasureCache == null) mMeasureCache = new LongSparseLongArray(2);  
  
       final boolean forceLayout = (mPrivateFlags & PFLAG_FORCE_LAYOUT) == PFLAG_FORCE_LAYOUT;  
  
       // Optimize layout by avoiding an extra EXACTLY pass when the view is  
       // already measured as the correct size. In API 23 and below, this  
       // extra pass is required to make LinearLayout re-distribute weight.  
       final boolean specChanged = widthMeasureSpec != mOldWidthMeasureSpec  
               || heightMeasureSpec != mOldHeightMeasureSpec;  
       final boolean isSpecExactly = MeasureSpec.getMode(widthMeasureSpec) == MeasureSpec.EXACTLY  
               && MeasureSpec.getMode(heightMeasureSpec) == MeasureSpec.EXACTLY;  
       final boolean matchesSpecSize = getMeasuredWidth() == MeasureSpec.getSize(widthMeasureSpec)  
               && getMeasuredHeight() == MeasureSpec.getSize(heightMeasureSpec);  
       final boolean needsLayout = specChanged  
               && (sAlwaysRemeasureExactly || !isSpecExactly || !matchesSpecSize);  
  
       if (forceLayout || needsLayout) {  
           // first clears the measured dimension flag  
           mPrivateFlags &= ~PFLAG_MEASURED_DIMENSION_SET;  
  
           resolveRtlPropertiesIfNeeded();  
  
           int cacheIndex = forceLayout ? -1 : mMeasureCache.indexOfKey(key);  
           if (cacheIndex < 0 || sIgnoreMeasureCache) {  
               // measure ourselves, this should set the measured dimension flag back  
               onMeasure(widthMeasureSpec, heightMeasureSpec);  
               mPrivateFlags3 &= ~PFLAG3_MEASURE_NEEDED_BEFORE_LAYOUT;  
           } else {  
               long value = mMeasureCache.valueAt(cacheIndex);  
               // Casting a long to int drops the high 32 bits, no mask needed  
               setMeasuredDimensionRaw((int) (value >> 32), (int) value);  
               mPrivateFlags3 |= PFLAG3_MEASURE_NEEDED_BEFORE_LAYOUT;  
           }  
  
           // flag not set, setMeasuredDimension() was not invoked, we raise  
           // an exception to warn the developer  
           if ((mPrivateFlags & PFLAG_MEASURED_DIMENSION_SET) != PFLAG_MEASURED_DIMENSION_SET) {  
               throw new IllegalStateException("View with id " + getId() + ": "  
                       + getClass().getName() + "#onMeasure() did not set the"  
                       + " measured dimension by calling"  
                       + " setMeasuredDimension()");  
           }  
  
           mPrivateFlags |= PFLAG_LAYOUT_REQUIRED;  
       }  
  
       mOldWidthMeasureSpec = widthMeasureSpec;  
       mOldHeightMeasureSpec = heightMeasureSpec;  
  
       mMeasureCache.put(key, ((long) mMeasuredWidth) << 32 |  
               (long) mMeasuredHeight & 0xffffffffL); // suppress sign extension  
   }  
```

> 这个方法真是又臭又长。。。讲道理的话其实我也看不懂， 但是我们只需要注意到一点， 就是这个方法调用了OnMeasure方法！
也就是说`measure` --> `OnMeasure`

 



`OnMeasure`就简单了 ：




    protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {  
          setMeasuredDimension(getDefaultSize(getSuggestedMinimumWidth(), widthMeasureSpec),  
                  getDefaultSize(getSuggestedMinimumHeight(), heightMeasureSpec));  
      } 

 
很简洁对不，但是简洁并不代表简单， 这里套了好几层。。。 不要被迷惑 ， 我们看最外层其实就是`setMeasureDimension()`.
设置宽和高， 这个宽和高是在 `getDefaultSize`方法里返回的， 所以我们来看看`getDefaultSize`的具体代码：



```
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
如果我们忽略掉`UNSPECIFIED`情况的话， 我们会发现第一个参数`size`根本用不到。。。
也就是说view的大小其实就是父view给他创建的`MeasureSpec`中的size大小。 

这也进一步说明，view在`wrap_content`情况下 ，大小还是会跟父view大小一样， 所以我们需要在自定义view的时候重写`OnMeasure`。








