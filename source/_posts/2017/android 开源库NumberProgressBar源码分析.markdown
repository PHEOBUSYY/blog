layout: "post"
title: "android 开源库NumberProgressBar源码分析"
date: "2017-01-12 15:12"
---

## android 开源库NumberProgressBar源码分析
  这个view是一个简单的自定义view来实现一个横向进度的效果,左边显示完成的进度条,右边显示未完成的进度条,中间是当前的当前的进度说明.实现的思路非常的简单,就是分别画这个view的3部分,完成+文字+未完成的.
  <!--more-->
  下面来看实现源码:

  核心的逻辑肯定是在 *onDraw* 方法中:
  ```java

    @Override
    protected void onDraw(Canvas canvas) {
        if (mIfDrawText) {
            calculateDrawRectF();
        } else {
            calculateDrawRectFWithoutProgressText();
        }

        if (mDrawReachedBar) {
            canvas.drawRect(mReachedRectF, mReachedBarPaint);
        }

        if (mDrawUnreachedBar) {
            canvas.drawRect(mUnreachedRectF, mUnreachedBarPaint);
        }

        if (mIfDrawText)
            canvas.drawText(mCurrentDrawText, mDrawTextStart, mDrawTextEnd, mTextPaint);
    }

  ```
  这里 *mIfDrawText* 表示是否显示进度百分比.
  然后通过 *mDrawReachedBar* , *mDrawUnreachedBar* , *mDrawUnreachedBar*  来控制完成部分,为完成部分,进度百分比的显示;
  核心计算在 *calculateDrawRectF* 中, *calculateDrawRectFWithoutProgressText* 方法和 *calculateDrawRectF* 类似,只是去掉了进度百分比文字的宽度计算.

  ```java
  private void calculateDrawRectF() {

       mCurrentDrawText = String.format("%d", getProgress() * 100 / getMax());
       mCurrentDrawText = mPrefix + mCurrentDrawText + mSuffix;
       mDrawTextWidth = mTextPaint.measureText(mCurrentDrawText);

       if (getProgress() == 0) {
           mDrawReachedBar = false;
           mDrawTextStart = getPaddingLeft();
       } else {
           mDrawReachedBar = true;
           mReachedRectF.left = getPaddingLeft();
           mReachedRectF.top = getHeight() / 2.0f - mReachedBarHeight / 2.0f;
           mReachedRectF.right = (getWidth() - getPaddingLeft() - getPaddingRight()) / (getMax() * 1.0f) * getProgress() - mOffset + getPaddingLeft();
           mReachedRectF.bottom = getHeight() / 2.0f + mReachedBarHeight / 2.0f;
           mDrawTextStart = (mReachedRectF.right + mOffset);
       }

       mDrawTextEnd = (int) ((getHeight() / 2.0f) - ((mTextPaint.descent() + mTextPaint.ascent()) / 2.0f));

       if ((mDrawTextStart + mDrawTextWidth) >= getWidth() - getPaddingRight()) {
           mDrawTextStart = getWidth() - getPaddingRight() - mDrawTextWidth;
           mReachedRectF.right = mDrawTextStart - mOffset;
       }

       float unreachedBarStart = mDrawTextStart + mDrawTextWidth + mOffset;
       if (unreachedBarStart >= getWidth() - getPaddingRight()) {
           mDrawUnreachedBar = false;
       } else {
           mDrawUnreachedBar = true;
           mUnreachedRectF.left = unreachedBarStart;
           mUnreachedRectF.right = getWidth() - getPaddingRight();
           mUnreachedRectF.top = getHeight() / 2.0f + -mUnreachedBarHeight / 2.0f;
           mUnreachedRectF.bottom = getHeight() / 2.0f + mUnreachedBarHeight / 2.0f;
       }
   }

  ```
  首先这个进度条是横线的,那么三个部分的高度都是定死的,就是 (view的高度 - 完成部分的高度)/2,非常的简单的,重要的是在算左右坐标.
  先看完成部分的坐标:
  ```java
     mReachedRectF.left = getPaddingLeft();
     ...
    mReachedRectF.right = (getWidth() - getPaddingLeft() - getPaddingRight()) / (getMax() * 1.0f) * getProgress() - mOffset + getPaddingLeft();
  ```
  左边是paddingLeft,这个没上面可说的,看右边
  显示计算一下在当前 progress 下所占用的宽度,就是把总长度 (getWidth() - getPaddingLeft() - getPaddingRight()) 除以 总的进度(getMax) 然后乘以当前的进度.
  然后加上左边的padding ,再减去 *mOffset* , 这个 *mOffset* 就是进度百分比文字与完成部分和未完成之间的间隔.防止文字和进度条紧挨着.

  然后计算出文字所要展示的横向坐标,mDrawTextStart就是完成部分的右边 + mOffset
  如果完成的部分的右坐标加上文字宽度之后超过了最大宽度,那么完成部分的右左边就是最大宽度减去paddingRight再减去文字的宽度
  文字所要展示的纵向左边也是固定的, 就是 高度/2 减去文字的偏移量/2

  剩下的是未完成的部分计算,先计算右边的开始坐标,就是文字的开始坐标+文字宽度+偏移量mOffset.
  如果未完成的开始部分超过最大限度了就不画了,把 *mDrawUnreachedBar* 设为false.如果没超过最大限度,那么未完成部分的开始就是前面的计算值,未完成部分的结束也很简单,就是最大宽度减去paddingRight.

  这里的计算非常的清晰简单,分分钟就可以明白了,如果不明白可以在纸上画一下就明白了.

### 参考资料
[github NumberProgressBar][9ca67ed2]

  [9ca67ed2]: https://github.com/daimajia/NumberProgressBar "github NumberProgressBar"
