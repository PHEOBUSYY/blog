layout: "post"
title: "android view滑动相关"
date: "2017-02-15 13:50"
---

## android view滑动相关
  在学习ViewDragHelper的过程中,用到了scroller来回弹到制定的位置,同时也很好奇scroller和computeScroll方法的关联,还有就是关于view坐标系一些api的学习等等,这些概念在这里说明一下.
  <!--more-->

### view的坐标系
![android view坐标](images/2017/02/android view坐标系.png)

通过上面的图片可以看的很明白了,getLeft,getRight,getTop,getBottom这四个方法是相对于父类边界的距离
在触摸事件中,getRawX,getRawY是当前触发点相对与屏幕的距离
而event中的getX,getY是当前触摸点相对于view本身的边界的距离.

这里还有个概念就是view本身的getX,getY不要和这个混淆了.
view本身的getX,getY是view的滑动距离加上getLeft
```java
public float getX() {
      return mLeft + getTranslationX();
  }
```
```java
public float getY() {
       return mTop + getTranslationY();
}
```
![view移动后坐标](images/2017/02/view移动后坐标.png)

### scroller的用法

首先要明白,让一个view从A点自己移动到B点并不是scroller做的,scroller本身只是用来负责计算的,也就是在你调用 *scroller.startScroll(int startX,int startY ,int distanceX,int distanceY)* 这个方法时候,本身只是在计算.

```java
public void startScroll(int startX, int startY, int dx, int dy, int duration) {
       mMode = SCROLL_MODE;
       mFinished = false;
       mDuration = duration;
       mStartTime = AnimationUtils.currentAnimationTimeMillis();
       mStartX = startX;
       mStartY = startY;
       mFinalX = startX + dx;
       mFinalY = startY + dy;
       mDeltaX = dx;
       mDeltaY = dy;
       mDurationReciprocal = 1.0f / (float) mDuration;
   }
```
在这个方法里面是没有关于view移动或者刷新的任何相关代码的.那到底scroller是怎么让view发生位移的呢.
奥妙就在我们需要在自定义的view中复写 *computeScroll* 方法.在 *computeScroll* 方法中通过调用 *scroller.computeScrollOffset* 方法来获取当前scroller的计算状态,如果该方法返回false,说明scroller仍然在计算,也就是说view可以继续移动还没有移动完成.反之返回ture说明计算完了,也就是这个时候view已经到了制定位置了.这个时候通过对 *computeScrollOffset* 的返回值判断我们来调用api来对view做真正的移动.
```java
@Override
   public void computeScroll() {
       if (scroller.computeScrollOffset()) {
           scrollTo(scroller.getCurrX(),scroller.getCurrY());
           invalidate();
       }
   }
```
可以看到,真正发生让view发生移动是这里的 *computeScrollOffset* 判断中的 *scrollTo* ,我们可以通过 scroller.getCurrX 和 getCurrY 来获取到当前计算到的坐标.

那另一个问题就是谁去调用的 *computeScroll* 方法呢?通过查看view的源码,在view的 *draw* 方法中可以看到调用了 *computeScroll* 方法,那这样就很容易明白了,为什么每次在调用 *scroller.startScroll* 方法之后,要紧接着调用 *invalid* 方法了.通过 *invalid* 方法来通知view重绘.然后就会调用 *computeScroll* 方法了.在这里完成改变view的位置的操作.

再说说什么情况下要用到 *scroller* ,个人理解就是在对view做滑动操作的时候,当你松开手指需要让view自己滚动到指定的位置的时候,可以交给scroller来完成自动滚动.so,一般要在touch事件的 *ACTION_UP* 中调用 *startScroll* 方法,紧接着调用 *invalid* 方法就可以了.

### scrollTo 和 scrollBy

通过 *scrollTo* 和 *scrollBy* 方法来是view发生位移是一种常规方式. 其中 *scrollTo* 是把view移动的绝对坐标的位置,而 *scrollBy* 是移动相对的位置.在源码中实际上 *scrollBy* 调用的就是 *scrollTo* 方法.

注意:这两个方法移动都是view的内容,如果viewGroup就是其内部的view.

getScrollX 和 getScrollY 方法返回对应的滚动距离,在复位的时候可以用到.

注意:在向右下方移动view的时候,应该传入的是负值的坐标

### view移动的7个方式
1. View.offsetLeftAndRight 和View.offsetTopAndBottom
2. scrollTo 和 ScrollBy
3. 修改Layout
4. 修改LayoutParams
5. 属性动画
6. 位移动画
