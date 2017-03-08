layout: "post"
title: "ViewDragHelper源码分析"
date: "2017-02-09 16:25"
---

## ViewDragHelper源码分析
在学习第三方的一个开源库的时候,发现了系统居然已为viewGroup控制的子view的手势移动提供了相关的组件,就是今天要介绍的 *ViewDragHelper* ,看了一下发现功能非常的强大,基本满足了我们平时的使用要求,首先我们先讲解一下它的用法,然后再从源码层面来分析一下它的实现思路.

### ViewDragHelper的用法
先通过一个简单的demo来看一下ViewDragHelper的用法:
```java
public class TestDragLayout extends LinearLayout {

    private ViewDragHelper mViewDragHelper;

    private View mDragView;

    public TestDragLayout(Context context) {
        this(context, null);
    }

    public TestDragLayout(Context context, AttributeSet attrs) {
        this(context, attrs, 0);
    }

    public TestDragLayout(Context context, AttributeSet attrs, int defStyleAttr) {
        super(context, attrs, defStyleAttr);
        init();
    }

    private void init() {
        mViewDragHelper = ViewDragHelper.create(this, 1f, new ViewDragCallBack());
    }

    private class ViewDragCallBack extends ViewDragHelper.Callback {

        @Override
        public boolean tryCaptureView(View child, int pointerId) {
            return mDragView.getId() == child.getId();
        }

        /**
         * 处理水平方向上的拖动
         *
         * @param child 拖动的View
         * @param left  移动到x轴的距离
         * @param dx    建议的移动的x距离
         */
        @Override
        public int clampViewPositionHorizontal(View child, int left, int dx) {
            //两个if主要是让view在ViewGroup中
            if (left < getPaddingLeft()) {
                return getPaddingLeft();
            }

            if (left > getWidth() - child.getMeasuredWidth()) {
                return getWidth() - child.getMeasuredWidth();
            }
            return left;
        }

        @Override
        public int clampViewPositionVertical(View child, int top, int dy) {
            if (top < getPaddingTop()) {
                return getPaddingTop();
            }

            if (top > getHeight() - child.getMeasuredHeight()) {
                return getHeight() - child.getMeasuredHeight();
            }

            return top;
        }

    }

    @Override
    public boolean onInterceptTouchEvent(MotionEvent ev) {
        return mViewDragHelper.shouldInterceptTouchEvent(ev);
    }

    @Override
    public boolean onTouchEvent(MotionEvent event) {
        mViewDragHelper.processTouchEvent(event);
        return true;
    }

    @Override
    protected void onFinishInflate() {
        super.onFinishInflate();
        mDragView = findViewById(R.id.dragview);
    }
}
```
我们先继承一个LinearLayout,然后在构造函数中调用 *init* 方法,在里面创建 *ViewDragHelper* 实例.然后在LinearLayout里面放入一个view,这个view就是我们要拖动的mDragView.运行程序,可以看到在布局中的view是可以在里面随便拖动的,同时也不会被拖出边界外面.是不是很简单呢?下面来分析一下使用方法:

使用ViewDragHelper需要三个步骤:
1. 创建ViewDragHelper实例
2. 触摸相关的方法调用,主要包括 *shouldInterceptTouchEvent(MotionEvent ev)* *processTouchEvent(MotionEvent ev)* 这两个方法
3. ViewDragHelper.Callback实例的编写,用来完成各种事件的回调

(一) 创建ViewDragHelper实例
  ViewDragHelper提供了两个创建方法,分别对应:
  ```java
  public static ViewDragHelper create(ViewGroup forParent, Callback cb) {
       return new ViewDragHelper(forParent.getContext(), forParent, cb);
   }
  ```
  ```java
  public static ViewDragHelper create(ViewGroup forParent, float sensitivity, Callback cb) {
      final ViewDragHelper helper = create(forParent, cb);
      helper.mTouchSlop = (int) (helper.mTouchSlop * (1 / sensitivity));
      return helper;
  }
  ```
  两个方法的区别在于第二个方法提供了一个 *sensitivity* 参数,这个参数用来表示拖动触发的灵敏度,越大便是越灵敏.因为这里 *helper.mTouchSlop* 是通过 *ViewConfiguration* 来获得当前设备的最小的触发距离的,距离越小表示越灵敏.
  ```java
  private ViewDragHelper(Context context, ViewGroup forParent, Callback cb) {
       if (forParent == null) {
           throw new IllegalArgumentException("Parent view may not be null");
       }
       if (cb == null) {
           throw new IllegalArgumentException("Callback may not be null");
       }

       mParentView = forParent;
       mCallback = cb;

       final ViewConfiguration vc = ViewConfiguration.get(context);
       final float density = context.getResources().getDisplayMetrics().density;
       mEdgeSize = (int) (EDGE_SIZE * density + 0.5f);

       mTouchSlop = vc.getScaledTouchSlop();
       mMaxVelocity = vc.getScaledMaximumFlingVelocity();
       mMinVelocity = vc.getScaledMinimumFlingVelocity();
       mScroller = ScrollerCompat.create(context, sInterpolator);
   }
  ```
(二) 触摸相关方法调用
  可以看到在继承的LinearLayout中,我们复写了两个触摸事件相关的方法:
  ```java
    @Override
     public boolean onInterceptTouchEvent(MotionEvent ev) {
         return mViewDragHelper.shouldInterceptTouchEvent(ev);
     }
  ```
  ```java
     @Override
     public boolean onTouchEvent(MotionEvent event) {
         mViewDragHelper.processTouchEvent(event);
         return true;
     }
  ```
  首先触摸事件会触发 *onInterceptTouchEvent* 如果该方法返回true,则表明当前的viewGroup要拦截该触摸事件,那么触摸事件就不会传递给下层的子类view.而是交由自己的 *onTouchEvent* 方法来处理. 而如果 *onInterceptTouchEvent* 方法返回false,则事件会传递给子类的 *onTouchEvent* 方法,如果子类view的 *onTouchEvent* 什么都没做返回false的话,事件会再次回到viewGroup的 *onTouchEvent* 方法来处理,反之事件被成功消化,不会回到上层的viewGroup了.这是android触摸事件的传递流程.还有一个 *dispatchTouchEvent* 方法来决定是否要分发触摸事件,事件的传递会先进入这个方法,然后在这个方法中通过判断 *onInterceptTouchEvent* 来决定是否要分发事件.

  回头来看这里的用到的触摸回调方法,先是在 *onInterceptTouchEvent* 方法,通过 *mViewDragHelper.shouldInterceptTouchEvent(ev)* 来决定是否分发事件给子view,如果这里返回true,就会进入 *onTouchEvent* 在里面调用 *mViewDragHelper.processTouchEvent(event)* 这个方法就是用来移动view的核心方法了.

  我们先来看 *shouldInterceptTouchEvent* 方法:
  ```java
  public boolean shouldInterceptTouchEvent(MotionEvent ev) {
        final int action = MotionEventCompat.getActionMasked(ev);
        final int actionIndex = MotionEventCompat.getActionIndex(ev);

        if (action == MotionEvent.ACTION_DOWN) {
            // Reset things for a new event stream, just in case we didn't get
            // the whole previous stream.
            cancel();
        }

        if (mVelocityTracker == null) {
            mVelocityTracker = VelocityTracker.obtain();
        }
        mVelocityTracker.addMovement(ev);

        switch (action) {
            case MotionEvent.ACTION_DOWN: {
                final float x = ev.getX();
                final float y = ev.getY();
                final int pointerId = ev.getPointerId(0);
                saveInitialMotion(x, y, pointerId);

                final View toCapture = findTopChildUnder((int) x, (int) y);

                // Catch a settling view if possible.
                if (toCapture == mCapturedView && mDragState == STATE_SETTLING) {
                    tryCaptureViewForDrag(toCapture, pointerId);
                }

                final int edgesTouched = mInitialEdgesTouched[pointerId];
                if ((edgesTouched & mTrackingEdges) != 0) {
                    mCallback.onEdgeTouched(edgesTouched & mTrackingEdges, pointerId);
                }
                break;
            }

            case MotionEventCompat.ACTION_POINTER_DOWN: {
                final int pointerId = ev.getPointerId(actionIndex);
                final float x = ev.getX(actionIndex);
                final float y = ev.getY(actionIndex);

                saveInitialMotion(x, y, pointerId);

                // A ViewDragHelper can only manipulate one view at a time.
                if (mDragState == STATE_IDLE) {
                    final int edgesTouched = mInitialEdgesTouched[pointerId];
                    if ((edgesTouched & mTrackingEdges) != 0) {
                        mCallback.onEdgeTouched(edgesTouched & mTrackingEdges, pointerId);
                    }
                } else if (mDragState == STATE_SETTLING) {
                    // Catch a settling view if possible.
                    final View toCapture = findTopChildUnder((int) x, (int) y);
                    if (toCapture == mCapturedView) {
                        tryCaptureViewForDrag(toCapture, pointerId);
                    }
                }
                break;
            }

            case MotionEvent.ACTION_MOVE: {
                if (mInitialMotionX == null || mInitialMotionY == null) break;

                // First to cross a touch slop over a draggable view wins. Also report edge drags.
                final int pointerCount = ev.getPointerCount();
                for (int i = 0; i < pointerCount; i++) {
                    final int pointerId = ev.getPointerId(i);

                    // If pointer is invalid then skip the ACTION_MOVE.
                    if (!isValidPointerForActionMove(pointerId)) continue;

                    final float x = ev.getX(i);
                    final float y = ev.getY(i);
                    final float dx = x - mInitialMotionX[pointerId];
                    final float dy = y - mInitialMotionY[pointerId];

                    final View toCapture = findTopChildUnder((int) x, (int) y);
                    final boolean pastSlop = toCapture != null && checkTouchSlop(toCapture, dx, dy);
                    if (pastSlop) {
                        // check the callback's
                        // getView[Horizontal|Vertical]DragRange methods to know
                        // if you can move at all along an axis, then see if it
                        // would clamp to the same value. If you can't move at
                        // all in every dimension with a nonzero range, bail.
                        final int oldLeft = toCapture.getLeft();
                        final int targetLeft = oldLeft + (int) dx;
                        final int newLeft = mCallback.clampViewPositionHorizontal(toCapture,
                                targetLeft, (int) dx);
                        final int oldTop = toCapture.getTop();
                        final int targetTop = oldTop + (int) dy;
                        final int newTop = mCallback.clampViewPositionVertical(toCapture, targetTop,
                                (int) dy);
                        final int horizontalDragRange = mCallback.getViewHorizontalDragRange(
                                toCapture);
                        final int verticalDragRange = mCallback.getViewVerticalDragRange(toCapture);
                        if ((horizontalDragRange == 0 || horizontalDragRange > 0
                                && newLeft == oldLeft) && (verticalDragRange == 0
                                || verticalDragRange > 0 && newTop == oldTop)) {
                            break;
                        }
                    }
                    reportNewEdgeDrags(dx, dy, pointerId);
                    if (mDragState == STATE_DRAGGING) {
                        // Callback might have started an edge drag
                        break;
                    }

                    if (pastSlop && tryCaptureViewForDrag(toCapture, pointerId)) {
                        break;
                    }
                }
                saveLastMotion(ev);
                break;
            }

            case MotionEventCompat.ACTION_POINTER_UP: {
                final int pointerId = ev.getPointerId(actionIndex);
                clearMotionHistory(pointerId);
                break;
            }

            case MotionEvent.ACTION_UP:
            case MotionEvent.ACTION_CANCEL: {
                cancel();
                break;
            }
        }

        return mDragState == STATE_DRAGGING;
  }
  ```
  可以看到里面还是对触摸事件的那几种基本的类型分别做处理,我们知道事件的触发类型对应: ACTION_DOWN --> ACTION_MOVE --> ACTION_MOVE --> ACTION_UP .同时这里加入了对多点触摸的处理.在上面的 ACTION_DOWN 的判断中,如果当前通过 *findTopChildUnder* 捕获的view就是之前的移动的view,并且处于释放状态,就重新捕获该view并调整状态.这种情况对应快速拖动之后松开后view会自己滑动一些距离的情况.第一次拖动的时候不会触发.

  下面进入 ACTION_MOVE ,在这里看到顺序获取多个触摸点,如果有没有越界,如果没有问题的话就会 调用 *tryCaptureViewForDrag* 来捕获要滑动的view,并求改其状态.
  ```java
  boolean tryCaptureViewForDrag(View toCapture, int pointerId) {
         if (toCapture == mCapturedView && mActivePointerId == pointerId) {
             // Already done!
             return true;
         }
         if (toCapture != null && mCallback.tryCaptureView(toCapture, pointerId)) {
             mActivePointerId = pointerId;
             captureChildView(toCapture, pointerId);
             return true;
         }
         return false;
     }
  ```
  该方法又会调用 *captureChildView* 方法:
  ```java
  public void captureChildView(View childView, int activePointerId) {
       if (childView.getParent() != mParentView) {
           throw new IllegalArgumentException("captureChildView: parameter must be a descendant "
                   + "of the ViewDragHelper's tracked parent view (" + mParentView + ")");
       }

       mCapturedView = childView;
       mActivePointerId = activePointerId;
       mCallback.onViewCaptured(childView, activePointerId);
       setDragState(STATE_DRAGGING);
   }
  ```
  在这个方法中最后修改view的状态为 *STATE_DRAGGING* .回到上面的 *shouldInterceptTouchEvent* 看最后一行的返回条件判断 *return mDragState == STATE_DRAGGING;* 正好对应这里的修改状态.也就是说当我们手势移动的时候,这里就会认为我们在移动触摸点下面的view,并返回true,方法调用就会进入下面要将的 *processTouchEvent* 方法了.

  ```java
  public void processTouchEvent(MotionEvent ev) {
        final int action = MotionEventCompat.getActionMasked(ev);
        final int actionIndex = MotionEventCompat.getActionIndex(ev);

        if (action == MotionEvent.ACTION_DOWN) {
            // Reset things for a new event stream, just in case we didn't get
            // the whole previous stream.
            cancel();
        }

        if (mVelocityTracker == null) {
            mVelocityTracker = VelocityTracker.obtain();
        }
        mVelocityTracker.addMovement(ev);

        switch (action) {
            case MotionEvent.ACTION_DOWN: {
                final float x = ev.getX();
                final float y = ev.getY();
                final int pointerId = ev.getPointerId(0);
                final View toCapture = findTopChildUnder((int) x, (int) y);

                saveInitialMotion(x, y, pointerId);

                // Since the parent is already directly processing this touch event,
                // there is no reason to delay for a slop before dragging.
                // Start immediately if possible.
                tryCaptureViewForDrag(toCapture, pointerId);

                final int edgesTouched = mInitialEdgesTouched[pointerId];
                if ((edgesTouched & mTrackingEdges) != 0) {
                    mCallback.onEdgeTouched(edgesTouched & mTrackingEdges, pointerId);
                }
                break;
            }

            case MotionEventCompat.ACTION_POINTER_DOWN: {
                final int pointerId = ev.getPointerId(actionIndex);
                final float x = ev.getX(actionIndex);
                final float y = ev.getY(actionIndex);

                saveInitialMotion(x, y, pointerId);

                // A ViewDragHelper can only manipulate one view at a time.
                if (mDragState == STATE_IDLE) {
                    // If we're idle we can do anything! Treat it like a normal down event.

                    final View toCapture = findTopChildUnder((int) x, (int) y);
                    tryCaptureViewForDrag(toCapture, pointerId);

                    final int edgesTouched = mInitialEdgesTouched[pointerId];
                    if ((edgesTouched & mTrackingEdges) != 0) {
                        mCallback.onEdgeTouched(edgesTouched & mTrackingEdges, pointerId);
                    }
                } else if (isCapturedViewUnder((int) x, (int) y)) {
                    // We're still tracking a captured view. If the same view is under this
                    // point, we'll swap to controlling it with this pointer instead.
                    // (This will still work if we're "catching" a settling view.)

                    tryCaptureViewForDrag(mCapturedView, pointerId);
                }
                break;
            }

            case MotionEvent.ACTION_MOVE: {
                if (mDragState == STATE_DRAGGING) {
                    // If pointer is invalid then skip the ACTION_MOVE.
                    if (!isValidPointerForActionMove(mActivePointerId)) break;

                    final int index = ev.findPointerIndex(mActivePointerId);
                    final float x = ev.getX(index);
                    final float y = ev.getY(index);
                    final int idx = (int) (x - mLastMotionX[mActivePointerId]);
                    final int idy = (int) (y - mLastMotionY[mActivePointerId]);

                    dragTo(mCapturedView.getLeft() + idx, mCapturedView.getTop() + idy, idx, idy);

                    saveLastMotion(ev);
                } else {
                    // Check to see if any pointer is now over a draggable view.
                    final int pointerCount = ev.getPointerCount();
                    for (int i = 0; i < pointerCount; i++) {
                        final int pointerId = ev.getPointerId(i);

                        // If pointer is invalid then skip the ACTION_MOVE.
                        if (!isValidPointerForActionMove(pointerId)) continue;

                        final float x = ev.getX(i);
                        final float y = ev.getY(i);
                        final float dx = x - mInitialMotionX[pointerId];
                        final float dy = y - mInitialMotionY[pointerId];

                        reportNewEdgeDrags(dx, dy, pointerId);
                        if (mDragState == STATE_DRAGGING) {
                            // Callback might have started an edge drag.
                            break;
                        }

                        final View toCapture = findTopChildUnder((int) x, (int) y);
                        if (checkTouchSlop(toCapture, dx, dy)
                                && tryCaptureViewForDrag(toCapture, pointerId)) {
                            break;
                        }
                    }
                    saveLastMotion(ev);
                }
                break;
            }

            case MotionEventCompat.ACTION_POINTER_UP: {
                final int pointerId = ev.getPointerId(actionIndex);
                if (mDragState == STATE_DRAGGING && pointerId == mActivePointerId) {
                    // Try to find another pointer that's still holding on to the captured view.
                    int newActivePointer = INVALID_POINTER;
                    final int pointerCount = ev.getPointerCount();
                    for (int i = 0; i < pointerCount; i++) {
                        final int id = ev.getPointerId(i);
                        if (id == mActivePointerId) {
                            // This one's going away, skip.
                            continue;
                        }

                        final float x = ev.getX(i);
                        final float y = ev.getY(i);
                        if (findTopChildUnder((int) x, (int) y) == mCapturedView
                                && tryCaptureViewForDrag(mCapturedView, id)) {
                            newActivePointer = mActivePointerId;
                            break;
                        }
                    }

                    if (newActivePointer == INVALID_POINTER) {
                        // We didn't find another pointer still touching the view, release it.
                        releaseViewForPointerUp();
                    }
                }
                clearMotionHistory(pointerId);
                break;
            }

            case MotionEvent.ACTION_UP: {
                if (mDragState == STATE_DRAGGING) {
                    releaseViewForPointerUp();
                }
                cancel();
                break;
            }

            case MotionEvent.ACTION_CANCEL: {
                if (mDragState == STATE_DRAGGING) {
                    dispatchViewReleased(0, 0);
                }
                cancel();
                break;
            }
        }
    }
  ```
  这个方法中也是对触摸事件情况的处理,其中的down和up和上面的 *shouldInterceptTouchEvent* 类似,重点就在 ACTION_MOVE 中,可以发现重点就在 *dragTo* 方法中:
  ```java
  private void dragTo(int left, int top, int dx, int dy) {
       int clampedX = left;
       int clampedY = top;
       final int oldLeft = mCapturedView.getLeft();
       final int oldTop = mCapturedView.getTop();
       if (dx != 0) {
           clampedX = mCallback.clampViewPositionHorizontal(mCapturedView, left, dx);
           ViewCompat.offsetLeftAndRight(mCapturedView, clampedX - oldLeft);
       }
       if (dy != 0) {
           clampedY = mCallback.clampViewPositionVertical(mCapturedView, top, dy);
           ViewCompat.offsetTopAndBottom(mCapturedView, clampedY - oldTop);
       }

       if (dx != 0 || dy != 0) {
           final int clampedDx = clampedX - oldLeft;
           final int clampedDy = clampedY - oldTop;
           mCallback.onViewPositionChanged(mCapturedView, clampedX, clampedY,
                   clampedDx, clampedDy);
       }
   }
  ```

  可以看到,在这里完成了view的位置移动处理.通过 *mCallback* 中的各个方法来获取移动范围,并且有个 *mCallback.onViewPositionChanged* 位置移动的回调. 下面讲一下 *callback* 的用法.

(三) ViewDragHelper.Callback的用法
  ```java
  public abstract static class Callback {
        /**
         * Called when the drag state changes. See the <code>STATE_*</code> constants
         * for more information.
         * 当view的拖拽状态改变时触发,对应下面写的三种情况中一种
         * @param state The new drag state
         *
         * @see #STATE_IDLE 当前没有被拖拽
         * @see #STATE_DRAGGING 正在别拖拽
         * @see #STATE_SETTLING 被拖拽后需要安置到一个位置中的状态
         */
        public void onViewDragStateChanged(int state) {}

        /**
         * Called when the captured view's position changes as the result of a drag or settle.
         * 当view在拖拽时位置发生变化时触发,对应上面的 dragTo 方法
         * @param changedView View whose position changed
         * @param left New X coordinate of the left edge of the view
         * @param top New Y coordinate of the top edge of the view
         * @param dx Change in X position from the last call
         * @param dy Change in Y position from the last call
         */
        public void onViewPositionChanged(View changedView, int left, int top, int dx, int dy) {}

        /**
         * Called when a child view is captured for dragging or settling. The ID of the pointer
         * currently dragging the captured view is supplied. If activePointerId is
         * identified as {@link #INVALID_POINTER} the capture is programmatic instead of
         * pointer-initiated.
         * 当一个view被捕获时触发
         * @param capturedChild Child view that was captured
         * @param activePointerId Pointer id tracking the child capture
         */
        public void onViewCaptured(View capturedChild, int activePointerId) {}

        /**
         * Called when the child view is no longer being actively dragged.
         * The fling velocity is also supplied, if relevant. The velocity values may
         * be clamped to system minimums or maximums.
         * 当拖拽动作释放时触发
         * <p>Calling code may decide to fling or otherwise release the view to let it
         * settle into place. It should do so using {@link #settleCapturedViewAt(int, int)}
         * or {@link #flingCapturedView(int, int, int, int)}. If the Callback invokes
         * one of these methods, the ViewDragHelper will enter {@link #STATE_SETTLING}
         * and the view capture will not fully end until it comes to a complete stop.
         * If neither of these methods is invoked before <code>onViewReleased</code> returns,
         * the view will stop in place and the ViewDragHelper will return to
         * {@link #STATE_IDLE}.</p>
         *
         * @param releasedChild The captured child view now being released
         * @param xvel X velocity of the pointer as it left the screen in pixels per second.
         * @param yvel Y velocity of the pointer as it left the screen in pixels per second.
         */
        public void onViewReleased(View releasedChild, float xvel, float yvel) {}

        /**
         * Called when one of the subscribed edges in the parent view has been touched
         * by the user while no child view is currently captured.
         * 当触发了viewGroup的边缘时触发
         * @param edgeFlags A combination of edge flags describing the edge(s) currently touched
         * @param pointerId ID of the pointer touching the described edge(s)
         * @see #EDGE_LEFT
         * @see #EDGE_TOP
         * @see #EDGE_RIGHT
         * @see #EDGE_BOTTOM
         */
        public void onEdgeTouched(int edgeFlags, int pointerId) {}

        /**
         * Called when the given edge may become locked. This can happen if an edge drag
         * was preliminarily rejected before beginning, but after {@link #onEdgeTouched(int, int)}
         * was called. This method should return true to lock this edge or false to leave it
         * unlocked. The default behavior is to leave edges unlocked.
         * 是否锁定边缘的触摸
         * @param edgeFlags A combination of edge flags describing the edge(s) locked
         * @return true to lock the edge, false to leave it unlocked
         */
        public boolean onEdgeLock(int edgeFlags) {
            return false;
        }

        /**
         * Called when the user has started a deliberate drag away from one
         * of the subscribed edges in the parent view while no child view is currently captured.
         * 边缘触摸开始时触发
         * @param edgeFlags A combination of edge flags describing the edge(s) dragged
         * @param pointerId ID of the pointer touching the described edge(s)
         * @see #EDGE_LEFT
         * @see #EDGE_TOP
         * @see #EDGE_RIGHT
         * @see #EDGE_BOTTOM
         */
        public void onEdgeDragStarted(int edgeFlags, int pointerId) {}

        /**
         * Called to determine the Z-order of child views.
         * 在寻找当前的触摸点下的view时会调用这个方法,比如两个子view叠加在一起之后,如果你想获得下面的那个时,可以改写这个方法.
         * @param index the ordered position to query for
         * @return index of the view that should be ordered at position <code>index</code>
         */
        public int getOrderedChildIndex(int index) {
            return index;
        }

        /**
         * Return the magnitude of a draggable child view's horizontal range of motion in pixels.
         * This method should return 0 for views that cannot move horizontally.
         * 获取被拖拽view的水平移动范围
         * @param child Child view to check
         * @return range of horizontal motion in pixels
         */
        public int getViewHorizontalDragRange(View child) {
            return 0;
        }

        /**
         * Return the magnitude of a draggable child view's vertical range of motion in pixels.
         * This method should return 0 for views that cannot move vertically.
         * 获取被拖拽view的垂直移动范围
         * @param child Child view to check
         * @return range of vertical motion in pixels
         */
        public int getViewVerticalDragRange(View child) {
            return 0;
        }

        /**
         * Called when the user's input indicates that they want to capture the given child view
         * with the pointer indicated by pointerId. The callback should return true if the user
         * is permitted to drag the given view with the indicated pointer.
         *
         * <p>ViewDragHelper may call this method multiple times for the same view even if
         * the view is already captured; this indicates that a new pointer is trying to take
         * control of the view.</p>
         * 尝试捕获当前触摸的view
         * <p>If this method returns true, a call to {@link #onViewCaptured(android.view.View, int)}
         * will follow if the capture is successful.</p>
         *
         * @param child Child the user is attempting to capture
         * @param pointerId ID of the pointer attempting the capture
         * @return true if capture should be allowed, false otherwise
         */
        public abstract boolean tryCaptureView(View child, int pointerId);

        /**
         * Restrict the motion of the dragged child view along the horizontal axis.
         * The default implementation does not allow horizontal motion; the extending
         * class must override this method and provide the desired clamping.
         * 限制水平方向的移动范围
         *
         * @param child Child view being dragged
         * @param left Attempted motion along the X axis
         * @param dx Proposed change in position for left
         * @return The new clamped position for left
         */
        public int clampViewPositionHorizontal(View child, int left, int dx) {
            return 0;
        }

        /**
         * Restrict the motion of the dragged child view along the vertical axis.
         * The default implementation does not allow vertical motion; the extending
         * class must override this method and provide the desired clamping.
         * 限制垂直方向的移动范围
         *
         * @param child Child view being dragged
         * @param top Attempted motion along the Y axis
         * @param dy Proposed change in position for top
         * @return The new clamped position for top
         */
        public int clampViewPositionVertical(View child, int top, int dy) {
            return 0;
        }
    }
  ```

  ViewDragHelper的内部流程实现就讲完了,有些细节这里就不展开了,感兴趣的同学可以读一下源码.在额外补充ViewDragHelper的常用方法 *settleCapturedViewAt* :
  ```java
  public boolean settleCapturedViewAt(int finalLeft, int finalTop) {
         if (!mReleaseInProgress) {
             throw new IllegalStateException("Cannot settleCapturedViewAt outside of a call to "
                     + "Callback#onViewReleased");
         }

         return forceSettleCapturedViewAt(finalLeft, finalTop,
                 (int) VelocityTrackerCompat.getXVelocity(mVelocityTracker, mActivePointerId),
                 (int) VelocityTrackerCompat.getYVelocity(mVelocityTracker, mActivePointerId));
     }
  ```
  这个方法,用来直接把拖动的view放在指定的位置上.
  ```java
  private boolean forceSettleCapturedViewAt(int finalLeft, int finalTop, int xvel, int yvel) {
       final int startLeft = mCapturedView.getLeft();
       final int startTop = mCapturedView.getTop();
       final int dx = finalLeft - startLeft;
       final int dy = finalTop - startTop;

       if (dx == 0 && dy == 0) {
           // Nothing to do. Send callbacks, be done.
           mScroller.abortAnimation();
           setDragState(STATE_IDLE);
           return false;
       }

       final int duration = computeSettleDuration(mCapturedView, dx, dy, xvel, yvel);
       mScroller.startScroll(startLeft, startTop, dx, dy, duration);

       setDragState(STATE_SETTLING);
       return true;
   }
  ```
  可以看到这里通过Scroller来完成view的平滑移动的.这个方法 *settleCapturedViewAt* 在拖动view释放之后让view进入指定位置的时候会非常有用.
  注意:一定要在viewGroup调用如下方法来完成view的平滑移动,在调用 *settleCapturedViewAt* 方法的时候.
  ```java
  @Override
  public void computeScroll() {
      super.computeScroll();
      if (mDragHelper.continueSettling(true)) {
          ViewCompat.postInvalidateOnAnimation(this);
      }
  }
  ```
