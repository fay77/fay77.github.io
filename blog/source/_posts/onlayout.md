---
title: 自定义View之OnLayout过程浅析
date: 2018-05-05 13:32:54
tags:
---
<!--more-->

> 直接看view的layout源码：



```
public void layout(int l, int t, int r, int b) {  
       if ((mPrivateFlags3 & PFLAG3_MEASURE_NEEDED_BEFORE_LAYOUT) != 0) {  
           onMeasure(mOldWidthMeasureSpec, mOldHeightMeasureSpec);  
           mPrivateFlags3 &= ~PFLAG3_MEASURE_NEEDED_BEFORE_LAYOUT;  
       }  
  
       int oldL = mLeft;  
       int oldT = mTop;  
       int oldB = mBottom;  
       int oldR = mRight;  
  
       boolean changed = isLayoutModeOptical(mParent) ?  
               setOpticalFrame(l, t, r, b) : setFrame(l, t, r, b);  
  
       if (changed || (mPrivateFlags & PFLAG_LAYOUT_REQUIRED) == PFLAG_LAYOUT_REQUIRED) {  
           onLayout(changed, l, t, r, b);  
           mPrivateFlags &= ~PFLAG_LAYOUT_REQUIRED;  
  
           ListenerInfo li = mListenerInfo;  
           if (li != null && li.mOnLayoutChangeListeners != null) {  
               ArrayList<OnLayoutChangeListener> listenersCopy =  
                       (ArrayList<OnLayoutChangeListener>)li.mOnLayoutChangeListeners.clone();  
               int numListeners = listenersCopy.size();  
               for (int i = 0; i < numListeners; ++i) {  
                   listenersCopy.get(i).onLayoutChange(this, l, t, r, b, oldL, oldT, oldR, oldB);  
               }  
           }  
       }  
  
       mPrivateFlags &= ~PFLAG_FORCE_LAYOUT;  
       mPrivateFlags3 |= PFLAG3_IS_LAID_OUT;  
   }  
```


----------


**这里挑重点来看** 


```
boolean changed = isLayoutModeOptical(mParent) ?  
             setOpticalFrame(l, t, r, b) : setFrame(l, t, r, b);  




private boolean setOpticalFrame(int left, int top, int right, int bottom) {  
        Insets parentInsets = mParent instanceof View ?  
                ((View) mParent).getOpticalInsets() : Insets.NONE;  
        Insets childInsets = getOpticalInsets();  
        return setFrame(  
                left   + parentInsets.left - childInsets.left,  
                top    + parentInsets.top  - childInsets.top,  
                right  + parentInsets.left + childInsets.right,  
                bottom + parentInsets.top  + childInsets.bottom);  
    }  
```

> 点进`setOpticalFrame`我们发现最终也是调用的`setFrame`方法， 所以我们直接来看这个方法：

```
protected boolean setFrame(int left, int top, int right, int bottom) {  
      boolean changed = false;  
  
      if (DBG) {  
          Log.d("View", this + " View.setFrame(" + left + "," + top + ","  
                  + right + "," + bottom + ")");  
      }  
  
      if (mLeft != left || mRight != right || mTop != top || mBottom != bottom) {  
          changed = true;  
  
          // Remember our drawn bit  
          int drawn = mPrivateFlags & PFLAG_DRAWN;  
  
          int oldWidth = mRight - mLeft;  
          int oldHeight = mBottom - mTop;  
          int newWidth = right - left;  
          int newHeight = bottom - top;  
          boolean sizeChanged = (newWidth != oldWidth) || (newHeight != oldHeight);  
  
          // Invalidate our old position  
          invalidate(sizeChanged);  
  
          mLeft = left;  
          mTop = top;  
          mRight = right;  
          mBottom = bottom;  
          mRenderNode.setLeftTopRightBottom(mLeft, mTop, mRight, mBottom);  
  
          mPrivateFlags |= PFLAG_HAS_BOUNDS;  
  
  
          if (sizeChanged) {  
              sizeChange(newWidth, newHeight, oldWidth, oldHeight);  
          }  
  
          if ((mViewFlags & VISIBILITY_MASK) == VISIBLE || mGhostView != null) {  
              // If we are visible, force the DRAWN bit to on so that  
              // this invalidate will go through (at least to our parent).  
              // This is because someone may have invalidated this view  
              // before this call to setFrame came in, thereby clearing  
              // the DRAWN bit.  
              mPrivateFlags |= PFLAG_DRAWN;  
              invalidate(sizeChanged);  
              // parent display list may need to be recreated based on a change in the bounds  
              // of any child  
              invalidateParentCaches();  
          }  
  
          // Reset drawn bit to original value (invalidate turns it off)  
          mPrivateFlags |= drawn;  
  
          mBackgroundSizeChanged = true;  
          if (mForegroundInfo != null) {  
              mForegroundInfo.mBoundsChanged = true;  
          }  
  
          notifySubtreeAccessibilityStateChangedIfNeeded();  
      }  
      return changed;  
  }  
```

**看到这句：**



    if (mLeft != left || mRight != right || mTop != top || mBottom != bottom)   
              changed = true; 

 

在该方法中把l，t， r， b分别与之前的`mLeft`，`mTop`，`mRight`，`mBottom`一一作比较，假若其中任意一个值发生了变化，那么就判定该View的位置发生了变化 



    if (changed || (mPrivateFlags & PFLAG_LAYOUT_REQUIRED) == PFLAG_LAYOUT_REQUIRED) {  
               onLayout(changed, l, t, r, b);

  


诺发生变化则会调用`onLayout`方法。




接着我们来看View的`onLayout`方法：



    protected void onLayout(boolean changed, int left, int top, int right, int bottom) {  
       }  

**居然是空的！**


查看注释我们发现`View`的`onLayout`是确定子`view`的位置的，所以我们直接来看`viewGroup`的`onLayout`方法：



    @Override  
       protected abstract void onLayout(boolean changed,  
               int l, int t, int r, int b);  

居然是个抽象方法！


到这里我们发现`view`和`viewGroup`都没有真正实现`onLayout`方法。



既然`ViewGroup`中的方法是抽象方法，那么子类就一定会重写这个方法， 我们来看`LinearLayout`：



    @Override  
       protected void onLayout(boolean changed, int l, int t, int r, int b) {  
           if (mOrientation == VERTICAL) {  
               layoutVertical(l, t, r, b);  
           } else {  
               layoutHorizontal(l, t, r, b);  
           }  
       } 

 


果然重写了，并且分为水平跟垂直的两种情况


随便挑一个来看看



```
void layoutVertical(int left, int top, int right, int bottom) {  
       final int paddingLeft = mPaddingLeft;  
  
       int childTop;  
       int childLeft;  
         
       // Where right end of child should go  
       final int width = right - left;  
       int childRight = width - mPaddingRight;  
         
       // Space available for child  
       int childSpace = width - paddingLeft - mPaddingRight;  
         
       final int count = getVirtualChildCount();  
  
       final int majorGravity = mGravity & Gravity.VERTICAL_GRAVITY_MASK;  
       final int minorGravity = mGravity & Gravity.RELATIVE_HORIZONTAL_GRAVITY_MASK;  
  
       switch (majorGravity) {  
          case Gravity.BOTTOM:  
              // mTotalLength contains the padding already  
              childTop = mPaddingTop + bottom - top - mTotalLength;  
              break;  
  
              // mTotalLength contains the padding already  
          case Gravity.CENTER_VERTICAL:  
              childTop = mPaddingTop + (bottom - top - mTotalLength) / 2;  
              break;  
  
          case Gravity.TOP:  
          default:  
              childTop = mPaddingTop;  
              break;  
       }  
  
       for (int i = 0; i < count; i++) {  
           final View child = getVirtualChildAt(i);  
           if (child == null) {  
               childTop += measureNullChild(i);  
           } else if (child.getVisibility() != GONE) {  
               final int childWidth = child.getMeasuredWidth();  
               final int childHeight = child.getMeasuredHeight();  
                 
               final LinearLayout.LayoutParams lp =  
                       (LinearLayout.LayoutParams) child.getLayoutParams();  
                 
               int gravity = lp.gravity;  
               if (gravity < 0) {  
                   gravity = minorGravity;  
               }  
               final int layoutDirection = getLayoutDirection();  
               final int absoluteGravity = Gravity.getAbsoluteGravity(gravity, layoutDirection);  
               switch (absoluteGravity & Gravity.HORIZONTAL_GRAVITY_MASK) {  
                   case Gravity.CENTER_HORIZONTAL:  
                       childLeft = paddingLeft + ((childSpace - childWidth) / 2)  
                               + lp.leftMargin - lp.rightMargin;  
                       break;  
  
                   case Gravity.RIGHT:  
                       childLeft = childRight - childWidth - lp.rightMargin;  
                       break;  
  
                   case Gravity.LEFT:  
                   default:  
                       childLeft = paddingLeft + lp.leftMargin;  
                       break;  
               }  
  
               if (hasDividerBeforeChildAt(i)) {  
                   childTop += mDividerHeight;  
               }  
  
               childTop += lp.topMargin;  
               setChildFrame(child, childLeft, childTop + getLocationOffset(child),  
                       childWidth, childHeight);  
               childTop += childHeight + lp.bottomMargin + getNextLocationOffset(child);  
  
               i += getChildrenSkipCount(child, i);  
           }  
       }  
   }  
```

> 这里简单的分析下`layoutVertical`的逻辑，
> 首先遍历所有子元素并调用`setChildFrame`方法为子元素指定对应的位置，其中`childTop`在不断的增大，这就意味着越后面的子元素位置就越靠下，刚好符合垂直`linearLayout`的特性。




----------




