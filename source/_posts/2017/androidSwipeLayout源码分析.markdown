layout: "post"
title: "androidSwipeLayout源码分析"
date: "2017-01-23 15:19"
---
## android SwipeLayout源码分析

swipeLayout是代码家写的一个支持手势滑动的开源库,初看的时候感觉特别惊艳,用户体验也非常的棒,特别好奇是怎么实现的,故抽时间研究了下.
通过分析代码结构得出swipeLayout主要分为三大部分:
1. 内部view初始设置
2. 内部ViewDragHelper的callback实现
3. 冲突解决与注意事项

<!--more-->
### 内部view设置

这里说的内部各个view实际上指的是swipeLayout的子view,查看源码第一行发现swipeLayout继承的是FrameLayout,我们知道,如果在FrameLayout中你不设置子view的gravity的话,它们会全部叠放在一起,其中在xml中最底部声明的view是在顶层展现,依次类推.咱们这里指的就是这些个view.

#### 内部view的类型

在swipeLayout中,作者把view分为两大类,顶层默认展示的view叫做 *surfaceView* ,通过滑动显示的下层的方向的4个view叫做 *bottomView* .滑动支持上下左右4个方向,所以这里bottomView放入一个view集合.对应下面的枚举和map集合:
```java
public enum DragEdge {
       Left,
       Top,
       Right,
       Bottom
   }
```
```java
 private LinkedHashMap<DragEdge, View> mDragEdges = new LinkedHashMap<>();
```
可以看到,这里通过 *DragEdge* 来对应上下左右4个方向的滑动,然后通过 *mDragEdges* 来存放4个方向对应的 *bottomView* .

#### 内部view的位置放置

我们知道关于ViewGroup中child的位置放置是在 *onLayout* 方法中配置的.so,这里我们看一下swipeLayout的 *onLayout* 方法:
```java
@Override
    protected void onLayout(boolean changed, int l, int t, int r, int b) {
        updateBottomViews();

        if (mOnLayoutListeners != null) for (int i = 0; i < mOnLayoutListeners.size(); i++) {
            mOnLayoutListeners.get(i).onLayout(this);
        }
    }
```
这里重点进入 *updateBottomViews* 方法,下面的那个位置回调先不用考虑:
```java
private void updateBottomViews() {
        View currentBottomView = getCurrentBottomView();
        if (currentBottomView != null) {
            if (mCurrentDragEdge == DragEdge.Left || mCurrentDragEdge == DragEdge.Right) {
                mDragDistance = currentBottomView.getMeasuredWidth() - dp2px(getCurrentOffset());
            } else {
                mDragDistance = currentBottomView.getMeasuredHeight() - dp2px(getCurrentOffset());
            }
        }
        //在这里对覆盖着的view做重新排列,按照需要
        if (mShowMode == ShowMode.PullOut) {
            layoutPullOut();
        } else if (mShowMode == ShowMode.LayDown) {
            layoutLayDown();
        }

        safeBottomView();
    }
```
这个 *updateBottomViews* 方法是swipeLayout中非常重要的一个方法,一定要明白其中的含义.下面来挨个说明一下:
  1. 首先是获取 *currentBottomView* 也就是当前滑动方式要显示的下层view.进入 *getCurrentBottomView* 方法:
    ```java
    public View getCurrentBottomView() {
        List<View> bottoms = getBottomViews();
        if (mCurrentDragEdge.ordinal() < bottoms.size()) {
            return bottoms.get(mCurrentDragEdge.ordinal());
        }
        return null;
    }
    ```
    在这个方法中,通过 *mCurrentDragEdge* 来获取到当前的滑动类型,然后从 *bottomView* 的集合中找到对应滑动类型的 *bottomView* .那么 *mCurrentDragEdge* 是在哪里配置的呢?
    ```java
    private static final DragEdge DefaultDragEdge = DragEdge.Right;

    private DragEdge mCurrentDragEdge = DefaultDragEdge;
    ```
    可以看到默认的 *mCurrentDragEdge* 是右侧滑动,除了初始化设置 *mCurrentDragEdge* 外,我们在代码中还可以发现一个方法 :
    ```java
    private void setCurrentDragEdge(DragEdge dragEdge) {
       mCurrentDragEdge = dragEdge;
       updateBottomViews();
    }
    ```
    这个方法用来设置滑动方式,再追踪这个方法可以发现是在touch方法中来根据touch的event参数来设置滑动方式的,这个后续讲到touch事件处理的时候再说.

    再来看如何通过 *getBottomViews* 方法来获取所有的 *bottomView* 的.
    ```java
    public List<View> getBottomViews() {
      ArrayList<View> bottoms = new ArrayList<View>();
      for (DragEdge dragEdge : DragEdge.values()) {
          bottoms.add(mDragEdges.get(dragEdge));
      }
      return bottoms;
    }
    ```
    里面通过遍历上面提到的 *mDragEdges* 来获取所有的 *bottomView*.那这个 *mDragEdges* 又是在哪里设置的呢.通过查看代码调用我们在 *addView* 方法中找到了下面的代码:
    ```java

    @Override
    public void addView(View child, int index, ViewGroup.LayoutParams params) {
        if (child == null) return;
        int gravity = Gravity.NO_GRAVITY;
        try {
            gravity = (Integer) params.getClass().getField("gravity").get(params);
        } catch (Exception e) {
            e.printStackTrace();
        }

        if (gravity > 0) {
            gravity = GravityCompat.getAbsoluteGravity(gravity, ViewCompat.getLayoutDirection(this));

            if ((gravity & Gravity.LEFT) == Gravity.LEFT) {
                mDragEdges.put(DragEdge.Left, child);
            }
            if ((gravity & Gravity.RIGHT) == Gravity.RIGHT) {
                mDragEdges.put(DragEdge.Right, child);
            }
            if ((gravity & Gravity.TOP) == Gravity.TOP) {
                mDragEdges.put(DragEdge.Top, child);
            }
            if ((gravity & Gravity.BOTTOM) == Gravity.BOTTOM) {
                mDragEdges.put(DragEdge.Bottom, child);
            }
        } else {
            for (Map.Entry<DragEdge, View> entry : mDragEdges.entrySet()) {
                if (entry.getValue() == null) {
                    //means used the drag_edge attr, the no gravity child should be use set
                    mDragEdges.put(entry.getKey(), child);
                    break;
                }
            }
        }
        if (child.getParent() == this) {
            return;
        }
        super.addView(child, index, params);
    }
    ```
    addView是所有的viewGroup都要调用的方法,在这里我们可以看到是通过 *gravity* 属性来配置到对应的滑动类型的.如果没有配置 *gravity* 属性,就会遍历 *mDragEdges* 依次塞入不同的滑动类型中,那 *mDragEdges* 的默认值又是在哪里配置的呢?通过作者的注释我们看到应该和xml中声明的属性有关,进入构造方法中找到了答案:
    ```java
    public SwipeLayout(Context context, AttributeSet attrs, int defStyle) {
       super(context, attrs, defStyle);
       mDragHelper = ViewDragHelper.create(this, mDragHelperCallback);
       mTouchSlop = ViewConfiguration.get(context).getScaledTouchSlop();

       TypedArray a = context.obtainStyledAttributes(attrs, R.styleable.SwipeLayout);
       int dragEdgeChoices = a.getInt(R.styleable.SwipeLayout_drag_edge, DRAG_RIGHT);
       mEdgeSwipesOffset[DragEdge.Left.ordinal()] = a.getDimension(R.styleable.SwipeLayout_leftEdgeSwipeOffset, 0);
       mEdgeSwipesOffset[DragEdge.Right.ordinal()] = a.getDimension(R.styleable.SwipeLayout_rightEdgeSwipeOffset, 0);
       mEdgeSwipesOffset[DragEdge.Top.ordinal()] = a.getDimension(R.styleable.SwipeLayout_topEdgeSwipeOffset, 0);
       mEdgeSwipesOffset[DragEdge.Bottom.ordinal()] = a.getDimension(R.styleable.SwipeLayout_bottomEdgeSwipeOffset, 0);
       setClickToClose(a.getBoolean(R.styleable.SwipeLayout_clickToClose, mClickToClose));

       if ((dragEdgeChoices & DRAG_LEFT) == DRAG_LEFT) {
           mDragEdges.put(DragEdge.Left, null);
       }
       if ((dragEdgeChoices & DRAG_TOP) == DRAG_TOP) {
           mDragEdges.put(DragEdge.Top, null);
       }
       if ((dragEdgeChoices & DRAG_RIGHT) == DRAG_RIGHT) {
           mDragEdges.put(DragEdge.Right, null);
       }
       if ((dragEdgeChoices & DRAG_BOTTOM) == DRAG_BOTTOM) {
           mDragEdges.put(DragEdge.Bottom, null);
       }
       int ordinal = a.getInt(R.styleable.SwipeLayout_show_mode, ShowMode.PullOut.ordinal());
       mShowMode = ShowMode.values()[ordinal];
       a.recycle();

   }
    ```
    可以看到这里,通过获取 *SwipeLayout_drag_edge* 属性来获取在xml中定义的drag类型,然后分别把对应的滑动类型放入 *mDragEdges* 中.如果在xml中没有声明,使用默认的滑动方式 *DRAG_RIGHT* 也就是从右侧滑动.

  2. 再来看 *updateBottomViews* 方法下面的部分,如何获取滑动的最大距离 *mDragDistance* :
      ```java
      if (currentBottomView != null) {
          if (mCurrentDragEdge == DragEdge.Left || mCurrentDragEdge == DragEdge.Right) {
              mDragDistance = currentBottomView.getMeasuredWidth() - dp2px(getCurrentOffset());
          } else {
              mDragDistance = currentBottomView.getMeasuredHeight() - dp2px(getCurrentOffset());
          }
      }
      ```
      这个 *mDragDistance* 关系到顶层surfaceView可以滑动的最大距离是多少.实际上就是 *currentBottomView* 的宽度或者高度加上偏移量.
  3. 再来看如何调整各个view的位置的
      ```java
      //在这里对覆盖着的view做重新排列,按照需要
      if (mShowMode == ShowMode.PullOut) {
          layoutPullOut();
      } else if (mShowMode == ShowMode.LayDown) {
          layoutLayDown();
      }
      ```
      这里对两种showMode做一下说明:
      a) pullout: 字面理解是拉出的意思,也就是说在拖拉上层view的时候,下层view是跟着拖拽出现的,比如你从左往右拖动上层的view,那下层的view也是通过从左往右出现的,也就是所谓的联动状态
      b) laydown: 字面解释是沉积和搁置的意思,也就是说在拖拽上层view的时候,下层view是不动的,下层view在上层拖动的时候慢慢展示.

      两种展示方式对应两种bottomView的放置方法, *layoutPullOut* 和 *layoutLayDown* 方法内部实现类似,这里只是讲一下第一个方法 :
      ```java
      void layoutPullOut() {
        View surfaceView = getSurfaceView();
        Rect surfaceRect = mViewBoundCache.get(surfaceView);
        if(surfaceRect == null) surfaceRect = computeSurfaceLayoutArea(false);
        if (surfaceView != null) {
            surfaceView.layout(surfaceRect.left, surfaceRect.top, surfaceRect.right, surfaceRect.bottom);
            //把这个view放在最前面
            bringChildToFront(surfaceView);
        }
        View currentBottomView = getCurrentBottomView();
        Rect bottomViewRect = mViewBoundCache.get(currentBottomView);
        //计算下层view的位置
        if(bottomViewRect == null) bottomViewRect = computeBottomLayoutAreaViaSurface(ShowMode.PullOut, surfaceRect);
        if (currentBottomView != null) {
            currentBottomView.layout(bottomViewRect.left, bottomViewRect.top, bottomViewRect.right, bottomViewRect.bottom);
        }
      }
      ```
      这个方法分成两部分,一部分是设置 *surfaceView* 的位置,一部分是设置 *currentBottomView* 的位置.同时把设置的位置参数存入缓存集合 *mViewBoundCache* 中.
      来看配置surfaceView位置的方法 *computeSurfaceLayoutArea* :
      ```java
      private Rect computeSurfaceLayoutArea(boolean open) {
        int l = getPaddingLeft(), t = getPaddingTop();
        if (open) {
            if (mCurrentDragEdge == DragEdge.Left)
                l = getPaddingLeft() + mDragDistance;
            else if (mCurrentDragEdge == DragEdge.Right)
                l = getPaddingLeft() - mDragDistance;
            else if (mCurrentDragEdge == DragEdge.Top)
                t = getPaddingTop() + mDragDistance;
            else t = getPaddingTop() - mDragDistance;
        }
        return new Rect(l, t, l + getMeasuredWidth(), t + getMeasuredHeight());
      }
      ```
      如果是覆盖状态,那么顶层view的上下左右应该和swipeLayout的上下左右相同.
      如果是打开状态,那么根据上下左右的滑动情况结合 *mDragDistance* 来调整位置,非常容易理解,如果看不明白代码的话可以在纸上划一下就明白了.

      再来看设置 *bottomView* 位置的方法 *computeBottomLayoutAreaViaSurface* :
      ```java
      private Rect computeBottomLayoutAreaViaSurface(ShowMode mode, Rect surfaceArea) {
       Rect rect = surfaceArea;
       View bottomView = getCurrentBottomView();

       int bl = rect.left, bt = rect.top, br = rect.right, bb = rect.bottom;
       if (mode == ShowMode.PullOut) {
           if (mCurrentDragEdge == DragEdge.Left)
               bl = rect.left - mDragDistance;
           else if (mCurrentDragEdge == DragEdge.Right)
               bl = rect.right;
           else if (mCurrentDragEdge == DragEdge.Top)
               bt = rect.top - mDragDistance;
           else bt = rect.bottom;

           if (mCurrentDragEdge == DragEdge.Left || mCurrentDragEdge == DragEdge.Right) {
               bb = rect.bottom;
               br = bl + (bottomView == null ? 0 : bottomView.getMeasuredWidth());
           } else {
               bb = bt + (bottomView == null ? 0 : bottomView.getMeasuredHeight());
               br = rect.right;
           }
       } else if (mode == ShowMode.LayDown) {
           if (mCurrentDragEdge == DragEdge.Left)
               br = bl + mDragDistance;
           else if (mCurrentDragEdge == DragEdge.Right)
               bl = br - mDragDistance;
           else if (mCurrentDragEdge == DragEdge.Top)
               bb = bt + mDragDistance;
           else bt = bb - mDragDistance;

       }
       return new Rect(bl, bt, br, bb);

      }
      ```
      这个方法直接根据不同的showMode来设置 *bottomView* 的位置.其实很好理解,如果是 *PullOut* 的话bottomView是在SwipeLayout的外部4个方向上,然后跟着滑动再慢慢进入swipeLayout中展示;而 *LayDown* 是bottomView本来就在swipeLayout中对应要显示的位置上,只是被上层view给盖住了而已.
  4. 最后来看 *safeBottomView* 方法:
      ```java
      private void safeBottomView() {
        Status status = getOpenStatus();
        List<View> bottoms = getBottomViews();

        if (status == Status.Close) {
            for (View bottom : bottoms) {
                if (bottom != null && bottom.getVisibility() != INVISIBLE) {
                    bottom.setVisibility(INVISIBLE);
                }
            }
        } else {
            View currentBottomView = getCurrentBottomView();
            if (currentBottomView != null && currentBottomView.getVisibility() != VISIBLE) {
                currentBottomView.setVisibility(VISIBLE);
            }
        }
      }
      ```
      通过判断当前顶层surfaceView的状态来控制bottomView的隐藏显示,如果是 *close* 状态,所有的bottomView都隐藏,反之显示当前滑动状态对应的bottomView.

####  ViewDragHelper相关配置
    进过上面的各类view的配置完成后,就进入view滑动的核心控制类ViewDragHelper的配置了,进过之前讲解 *ViewDragHelper的用法与源码分析* 我们明白想要让View可以被拖动的核心是实现ViewDragHelper中callback的各个接口方法.下面就来看一下swipeLayout中callback的实现.
    ```java
    private ViewDragHelper.Callback mDragHelperCallback = new ViewDragHelper.Callback() {

       @Override
       public int clampViewPositionHorizontal(View child, int left, int dx) {
           if (child == getSurfaceView()) {
               //顶层view可以滑动的最大横向距离
               switch (mCurrentDragEdge) {
                   case Top:
                   case Bottom:
                       return getPaddingLeft();
                   case Left:
                       if (left < getPaddingLeft()) return getPaddingLeft();
                       if (left > getPaddingLeft() + mDragDistance)
                           return getPaddingLeft() + mDragDistance;
                       break;
                   case Right:
                       if (left > getPaddingLeft()) return getPaddingLeft();
                       if (left < getPaddingLeft() - mDragDistance)
                           return getPaddingLeft() - mDragDistance;
                       break;
               }
           } else if (getCurrentBottomView() == child) {

               switch (mCurrentDragEdge) {
                   case Top:
                   case Bottom:
                       return getPaddingLeft();
                   case Left:
                       if (mShowMode == ShowMode.PullOut) {
                           if (left > getPaddingLeft()) return getPaddingLeft();
                       }
                       break;
                   case Right:
                       if (mShowMode == ShowMode.PullOut) {
                           if (left < getMeasuredWidth() - mDragDistance) {
                               return getMeasuredWidth() - mDragDistance;
                           }
                       }
                       break;
               }
           }
           return left;
       }

       @Override
       public int clampViewPositionVertical(View child, int top, int dy) {
           if (child == getSurfaceView()) {
               switch (mCurrentDragEdge) {
                   case Left:
                   case Right:
                       return getPaddingTop();
                   case Top:
                       if (top < getPaddingTop()) return getPaddingTop();
                       if (top > getPaddingTop() + mDragDistance)
                           return getPaddingTop() + mDragDistance;
                       break;
                   case Bottom:
                       if (top < getPaddingTop() - mDragDistance) {
                           return getPaddingTop() - mDragDistance;
                       }
                       if (top > getPaddingTop()) {
                           return getPaddingTop();
                       }
               }
           } else {
               View surfaceView = getSurfaceView();
               int surfaceViewTop = surfaceView == null ? 0 : surfaceView.getTop();
               switch (mCurrentDragEdge) {
                   case Left:
                   case Right:
                       return getPaddingTop();
                   case Top:
                       if (mShowMode == ShowMode.PullOut) {
                           if (top > getPaddingTop()) return getPaddingTop();
                       } else {
                           if (surfaceViewTop + dy < getPaddingTop())
                               return getPaddingTop();
                           if (surfaceViewTop + dy > getPaddingTop() + mDragDistance)
                               return getPaddingTop() + mDragDistance;
                       }
                       break;
                   case Bottom:
                       if (mShowMode == ShowMode.PullOut) {
                           if (top < getMeasuredHeight() - mDragDistance)
                               return getMeasuredHeight() - mDragDistance;
                       } else {
                           if (surfaceViewTop + dy >= getPaddingTop())
                               return getPaddingTop();
                           if (surfaceViewTop + dy <= getPaddingTop() - mDragDistance)
                               return getPaddingTop() - mDragDistance;
                       }
               }
           }
           return top;
       }

       @Override
       public boolean tryCaptureView(View child, int pointerId) {
           //这里相当于所有的子view都可以被滑动
           boolean result = child == getSurfaceView() || getBottomViews().contains(child);
           if (result) {
               isCloseBeforeDrag = getOpenStatus() == Status.Close;
           }
           return result;
       }

       @Override
       public int getViewHorizontalDragRange(View child) {
           return mDragDistance;
       }

       @Override
       public int getViewVerticalDragRange(View child) {
           return mDragDistance;
       }

       boolean isCloseBeforeDrag = true;

       @Override
       public void onViewReleased(View releasedChild, float xvel, float yvel) {
           super.onViewReleased(releasedChild, xvel, yvel);
           processHandRelease(xvel, yvel, isCloseBeforeDrag);
           for (SwipeListener l : mSwipeListeners) {
               l.onHandRelease(SwipeLayout.this, xvel, yvel);
           }

           invalidate();
       }

       @Override
       public void onViewPositionChanged(View changedView, int left, int top, int dx, int dy) {
           View surfaceView = getSurfaceView();
           if (surfaceView == null) return;
           View currentBottomView = getCurrentBottomView();
           int evLeft = surfaceView.getLeft(),
                   evRight = surfaceView.getRight(),
                   evTop = surfaceView.getTop(),
                   evBottom = surfaceView.getBottom();
           if (changedView == surfaceView) {
               //通知下层的view也移动对应的距离,因为你拖动上层view的时候,下层view是不会自己动的
               if (mShowMode == ShowMode.PullOut && currentBottomView != null) {
                   if (mCurrentDragEdge == DragEdge.Left || mCurrentDragEdge == DragEdge.Right) {
                       currentBottomView.offsetLeftAndRight(dx);
                   } else {
                       currentBottomView.offsetTopAndBottom(dy);
                   }
               }

           } else if (getBottomViews().contains(changedView)) {

               if (mShowMode == ShowMode.PullOut) {
                   surfaceView.offsetLeftAndRight(dx);
                   surfaceView.offsetTopAndBottom(dy);
               } else {
                   Rect rect = computeBottomLayDown(mCurrentDragEdge);
                   if (currentBottomView != null) {
                       //下层view保持不变
                       currentBottomView.layout(rect.left, rect.top, rect.right, rect.bottom);
                   }

                   int newLeft = surfaceView.getLeft() + dx, newTop = surfaceView.getTop() + dy;

                   if (mCurrentDragEdge == DragEdge.Left && newLeft < getPaddingLeft())
                       newLeft = getPaddingLeft();
                   else if (mCurrentDragEdge == DragEdge.Right && newLeft > getPaddingLeft())
                       newLeft = getPaddingLeft();
                   else if (mCurrentDragEdge == DragEdge.Top && newTop < getPaddingTop())
                       newTop = getPaddingTop();
                   else if (mCurrentDragEdge == DragEdge.Bottom && newTop > getPaddingTop())
                       newTop = getPaddingTop();
                   //上层的view移动
                   surfaceView.layout(newLeft, newTop, newLeft + getMeasuredWidth(), newTop + getMeasuredHeight());
               }
           }
           //回调
           dispatchRevealEvent(evLeft, evTop, evRight, evBottom);
           //回调
           dispatchSwipeEvent(evLeft, evTop, dx, dy);

           invalidate();
           //保持一下下层view的位置,方便layout的时候直接调用
           captureChildrenBound();
       }
    };
    ```
    我们来按照实现顺序重点讲解一下这几个接口方法:
    1. *tryCaptureView(View child, int pointerId)*
        这个方法用来判断当前手势滑动的view是否是我们允许滑动的view,如果返回false说明当前不允许这个view滑动,true的话是运行滑动.
        ```java
        @Override
       public boolean tryCaptureView(View child, int pointerId) {
           //这里相当于所有的子view都可以被滑动
           boolean result = child == getSurfaceView() || getBottomViews().contains(child);
           if (result) {
               isCloseBeforeDrag = getOpenStatus() == Status.Close;
           }
           return result;
        }
        ```
        在这里可以看到,所有的bottomView和surfaceView都是可以滑动的.
    2. *getViewHorizontalDragRange* 和 *getViewVerticalDragRange*
        这两个方法用来配置横向和纵向可以滑动的最大距离,这里统一用的就是上面计算出的 *mDragDistance*
        ```java
        @Override
        public int getViewHorizontalDragRange(View child) {
            return mDragDistance;
        }

        @Override
        public int getViewVerticalDragRange(View child) {
            return mDragDistance;
        }
        ```
    3. *clampViewPositionHorizontal* 和 *clampViewPositionVertical*
        这个方法用来控制横向和纵向允许滑动的最大距离,防止滑动越界.这里讲解一下横向的计算方式,纵向和这个类似就不重复了.
        ```java
        @Override
        public int clampViewPositionHorizontal(View child, int left, int dx) {
            if (child == getSurfaceView()) {
                //顶层view可以滑动的最大横向距离
                switch (mCurrentDragEdge) {
                    case Top:
                    case Bottom:
                        return getPaddingLeft();
                    case Left:
                        if (left < getPaddingLeft()) return getPaddingLeft();
                        if (left > getPaddingLeft() + mDragDistance)
                            return getPaddingLeft() + mDragDistance;
                        break;
                    case Right:
                        if (left > getPaddingLeft()) return getPaddingLeft();
                        if (left < getPaddingLeft() - mDragDistance)
                            return getPaddingLeft() - mDragDistance;
                        break;
                }
            } else if (getCurrentBottomView() == child) {

                switch (mCurrentDragEdge) {
                    case Top:
                    case Bottom:
                        return getPaddingLeft();
                    case Left:
                        if (mShowMode == ShowMode.PullOut) {
                            if (left > getPaddingLeft()) return getPaddingLeft();
                        }
                        break;
                    case Right:
                        if (mShowMode == ShowMode.PullOut) {
                            if (left < getMeasuredWidth() - mDragDistance) {
                                return getMeasuredWidth() - mDragDistance;
                            }
                        }
                        break;
                }
            }
            return left;
        }
        ```
        分两部分来看,首先是surfaceView
        如果是上下两个方向的滑动,那么相当于在横向上是不允许移动的,这里不设置为0是因为要考虑padding,所以返回left只能是getPaddingLeft.
        如果是横向滑动的话,又分为左右两个方向处理.
        如果drag类型是 *left* ,说明是向右滑动露出左边,这个left只能大于 *getPaddingLeft* 小于  *getPaddingLeft + mDragDistance* (正值).
        如果是 *right* ,说明向左拉动,露出右边,这个时候left只能是小于 *getPaddingLeft* 大于 *getPaddingLeft - mDragDistance* (负值).

        再来看bottomView
        上下两个方向不允许滑动,所以返回 *getPaddingLeft* .
        如果drag类型是 *left*
        a) pullout 这个时候bottomView是在swipeLayout的左侧外部,所以它可以滑动到的最大left只能是从负的到正的 *getPaddingLeft*
        如果drag类型是 *right*
        a) pullout 这个时候bottomView是在swipeLayout的右侧外部,所以它最多可以划进来自己本身宽度那么大的距离,所以这里用的 *getMeasuredWidth() - mDragDistance* ,用SwipeLayout的宽度减去bottomView的宽度,就是距离左边的距离.
    4. *onViewPositionChanged*
        ```java
        @Override
       public void onViewPositionChanged(View changedView, int left, int top, int dx, int dy) {
           View surfaceView = getSurfaceView();
           if (surfaceView == null) return;
           View currentBottomView = getCurrentBottomView();
           int evLeft = surfaceView.getLeft(),
                   evRight = surfaceView.getRight(),
                   evTop = surfaceView.getTop(),
                   evBottom = surfaceView.getBottom();
           if (changedView == surfaceView) {
               //通知下层的view也移动对应的距离,因为你拖动上层view的时候,下层view是不会自己动的
               if (mShowMode == ShowMode.PullOut && currentBottomView != null) {
                   if (mCurrentDragEdge == DragEdge.Left || mCurrentDragEdge == DragEdge.Right) {
                       currentBottomView.offsetLeftAndRight(dx);
                   } else {
                       currentBottomView.offsetTopAndBottom(dy);
                   }
               }

           } else if (getBottomViews().contains(changedView)) {

               if (mShowMode == ShowMode.PullOut) {
                   surfaceView.offsetLeftAndRight(dx);
                   surfaceView.offsetTopAndBottom(dy);
               } else {
                   Rect rect = computeBottomLayDown(mCurrentDragEdge);
                   if (currentBottomView != null) {
                       //下层view保持不变
                       currentBottomView.layout(rect.left, rect.top, rect.right, rect.bottom);
                   }

                   int newLeft = surfaceView.getLeft() + dx, newTop = surfaceView.getTop() + dy;

                   if (mCurrentDragEdge == DragEdge.Left && newLeft < getPaddingLeft())
                       newLeft = getPaddingLeft();
                   else if (mCurrentDragEdge == DragEdge.Right && newLeft > getPaddingLeft())
                       newLeft = getPaddingLeft();
                   else if (mCurrentDragEdge == DragEdge.Top && newTop < getPaddingTop())
                       newTop = getPaddingTop();
                   else if (mCurrentDragEdge == DragEdge.Bottom && newTop > getPaddingTop())
                       newTop = getPaddingTop();
                   //上层的view移动
                   surfaceView.layout(newLeft, newTop, newLeft + getMeasuredWidth(), newTop + getMeasuredHeight());
               }
           }
           //回调
           dispatchRevealEvent(evLeft, evTop, evRight, evBottom);
           //回调
           dispatchSwipeEvent(evLeft, evTop, dx, dy);

           invalidate();
           //保持一下下层view的位置,方便layout的时候直接调用
           captureChildrenBound();
        }
        ```
        这个方法是在滑动view的时候位置发生变化的回调方法,也是我们实现view滑动效果的关键.
        重点是对应 *pullout* 这种方式,因为是如果是这种显示模式的话,是要两个view联动,一个是ViewDragHelper帮我们滑动的那个view,另一个是要联动的另一个view.这个是联动的关键.
        先来看if判断第一部分,如果滑动的是surfaceView,让 *currentBottomView* 跟着surfaceView一起滑动相应的距离.
        再来看if判断的第二部分,如果滑动的是bottomView,这个时候要判断是否是pullout方式
          a) 是,那么也要保证在滑动bottomView的时候,surfaceView也要跟着移动对应的距离
          b) 不是,那么保证bottomView不要动也就是一直设置bottomView的固定位置,同时让surfaceView反向移动.
        最后来看剩下的几个方法:
          a) *dispatchRevealEvent* 和 *dispatchSwipeEvent* 这两个方法用来做滑动事件的回调,这里不做展开
          b) *invalidate* 重新要求swipeLayout绘制,实现滑动的关键方法
          c) *captureChildrenBound* 保存一下surfaceView和bottomView的位置信息,节省下次计算时间.

    5. onViewReleased
      ```java
      @Override
       public void onViewReleased(View releasedChild, float xvel, float yvel) {
           super.onViewReleased(releasedChild, xvel, yvel);
           processHandRelease(xvel, yvel, isCloseBeforeDrag);
           for (SwipeListener l : mSwipeListeners) {
               l.onHandRelease(SwipeLayout.this, xvel, yvel);
           }

           invalidate();
       }
      ```
      根据方法名称可以知道是用来处理手指松开之后的事件处理的.核心处理在 *processHandRelease* 方法.
      ```java
      protected void processHandRelease(float xvel, float yvel, boolean isCloseBeforeDragged) {
        float minVelocity = mDragHelper.getMinVelocity();
        View surfaceView = getSurfaceView();
        DragEdge currentDragEdge = mCurrentDragEdge;
        if (currentDragEdge == null || surfaceView == null) {
            return;
        }
        float willOpenPercent = (isCloseBeforeDragged ? mWillOpenPercentAfterClose : mWillOpenPercentAfterOpen);
        if (currentDragEdge == DragEdge.Left) {
            if (xvel > minVelocity) open();
            else if (xvel < -minVelocity) close();
            else {
                float openPercent = 1f * getSurfaceView().getLeft() / mDragDistance;
                if (openPercent > willOpenPercent) open();
                else close();
            }
        } else if (currentDragEdge == DragEdge.Right) {
            if (xvel > minVelocity) close();
            else if (xvel < -minVelocity) open();
            else {
                float openPercent = 1f * (-getSurfaceView().getLeft()) / mDragDistance;
                if (openPercent > willOpenPercent) open();
                else close();
            }
        } else if (currentDragEdge == DragEdge.Top) {
            if (yvel > minVelocity) open();
            else if (yvel < -minVelocity) close();
            else {
                float openPercent = 1f * getSurfaceView().getTop() / mDragDistance;
                if (openPercent > willOpenPercent) open();
                else close();
            }
        } else if (currentDragEdge == DragEdge.Bottom) {
            if (yvel > minVelocity) close();
            else if (yvel < -minVelocity) open();
            else {
                float openPercent = 1f * (-getSurfaceView().getTop()) / mDragDistance;
                if (openPercent > willOpenPercent) open();
                else close();
            }
        }
    }
    ```
    里面是大量的临界判断,比如当你拖拽的距离超过最大距离一半时松手时,那么应该进入到展开状态,反之就进入关闭状态,这是良好用户体验的基础.坐标判断就不展开了,来看最终的状态处理 *open* 和 *close* 方法.

    ```java
    public void close(boolean smooth, boolean notify) {
        View surface = getSurfaceView();
        if (surface == null) {
            return;
        }
        int dx, dy;
        if (smooth)
            mDragHelper.smoothSlideViewTo(getSurfaceView(), getPaddingLeft(), getPaddingTop());
        else {
            Rect rect = computeSurfaceLayoutArea(false);
            dx = rect.left - surface.getLeft();
            dy = rect.top - surface.getTop();
            surface.layout(rect.left, rect.top, rect.right, rect.bottom);
            if (notify) {
                dispatchRevealEvent(rect.left, rect.top, rect.right, rect.bottom);
                dispatchSwipeEvent(rect.left, rect.top, dx, dy);
            } else {
                safeBottomView();
            }
        }
        invalidate();
    }
    ```
    可以看到这里用到了 *smoothSlideViewTo* 这个方法,让surfaceView来复位,最后再重回swipeLayout.这里关于ViewDragHelper的 *smoothSlideViewTo* 方法有个关键点,在后面的注意事项中我们再说.
    ```java
    public void open(boolean smooth, boolean notify) {
       View surface = getSurfaceView(), bottom = getCurrentBottomView();
       if (surface == null) {
           return;
       }
       int dx, dy;
       Rect rect = computeSurfaceLayoutArea(true);
       if (smooth) {
           mDragHelper.smoothSlideViewTo(surface, rect.left, rect.top);
       } else {
           dx = rect.left - surface.getLeft();
           dy = rect.top - surface.getTop();
           surface.layout(rect.left, rect.top, rect.right, rect.bottom);
           if (getShowMode() == ShowMode.PullOut) {
               Rect bRect = computeBottomLayoutAreaViaSurface(ShowMode.PullOut, rect);
               if (bottom != null) {
                   bottom.layout(bRect.left, bRect.top, bRect.right, bRect.bottom);
               }
           }
           if (notify) {
               dispatchRevealEvent(rect.left, rect.top, rect.right, rect.bottom);
               dispatchSwipeEvent(rect.left, rect.top, dx, dy);
           } else {
               safeBottomView();
           }
       }
       invalidate();
    }
    ```
    open方法和close方法类似,也是通过 *smoothSlideViewTo* 来完成最终的展开的.

  到这里,callback的核心接口方法就讲完了,实际上swipeLayout的主要流程到这里就基本讲完了,可以看到主要实现就是callback中的逻辑处理.剩下的就是关于swipeLayout的一些注意事项和容易忽略的地方讲解.

####  冲突解决与注意事项
  到这里,你可能很好奇,为什么没有找到ViewDragHelper的调用地方,那是因为这里有两个关键点容易以往,在这里来重点讲解一下.先来看swipeLayout的触摸拦截方法 *onInterceptTouchEvent*
  ```java
  @Override
    public boolean onInterceptTouchEvent(MotionEvent ev) {
        if (!isSwipeEnabled()) {
            return false;
        }
        if (mClickToClose && getOpenStatus() == Status.Open && isTouchOnSurface(ev)) {
            return true;
        }
        for (SwipeDenier denier : mSwipeDeniers) {
            if (denier != null && denier.shouldDenySwipe(ev)) {
                return false;
            }
        }

        switch (ev.getAction()) {
            case MotionEvent.ACTION_DOWN:
                mDragHelper.processTouchEvent(ev);
                mIsBeingDragged = false;
                sX = ev.getRawX();
                sY = ev.getRawY();
                //if the swipe is in middle state(scrolling), should intercept the touch
                if (getOpenStatus() == Status.Middle) {
                    mIsBeingDragged = true;
                }
                break;
            case MotionEvent.ACTION_MOVE:
                boolean beforeCheck = mIsBeingDragged;
                checkCanDrag(ev);
                if (mIsBeingDragged) {
                    //向父类申请不要拦截触摸事件,注意这里是viewGroup向它的父类申请不要拦截触摸事件
                    ViewParent parent = getParent();
                    if (parent != null) {
                        parent.requestDisallowInterceptTouchEvent(true);
                    }
                }
                if (!beforeCheck && mIsBeingDragged) {
                    //let children has one chance to catch the touch, and request the swipe not intercept
                    //useful when swipeLayout wrap a swipeLayout or other gestural layout
                    return false;
                }
                break;

            case MotionEvent.ACTION_CANCEL:
            case MotionEvent.ACTION_UP:
                mIsBeingDragged = false;
                mDragHelper.processTouchEvent(ev);
                break;
            default://handle other action, such as ACTION_POINTER_DOWN/UP
                mDragHelper.processTouchEvent(ev);
        }
        return mIsBeingDragged;
    }
  ```
  在这里主要目的是为了解决如果和父类滑动事件冲突的时候如何处理呢,比如把swipeLayout放入listView,如果你沿着斜上方来滑动listView,很容易对swipeLayout造成影响,这个时候我们在 *ACTION_MOVE* 方法中,通过一个变量 *mIsBeingDragged* 判断是否要求父类不要拦截我们的swipeLayout的滑动事件.要求父类不要拦截滑动事件可以调用
  ```java
  ViewParent parent = getParent();
  if (parent != null) {
      parent.requestDisallowInterceptTouchEvent(true);
  }
  ```
  *mIsBeingDragged* 是通过 *checkCanDrag* 方法来赋值的.
  ```java
  //根据滑动的方式来判断是哪一种类型的drag方式
    private void checkCanDrag(MotionEvent ev) {
        if (mIsBeingDragged) return;
        if (getOpenStatus() == Status.Middle) {
            mIsBeingDragged = true;
            return;
        }
        Status status = getOpenStatus();
        float distanceX = ev.getRawX() - sX;
        float distanceY = ev.getRawY() - sY;
        float angle = Math.abs(distanceY / distanceX);
        angle = (float) Math.toDegrees(Math.atan(angle));
        if (getOpenStatus() == Status.Close) {
            //通过角度判断滑动的方向
            DragEdge dragEdge;
            if (angle < 45) {
                if (distanceX > 0 && isLeftSwipeEnabled()) {
                    dragEdge = DragEdge.Left;
                } else if (distanceX < 0 && isRightSwipeEnabled()) {
                    dragEdge = DragEdge.Right;
                } else return;

            } else {
                if (distanceY > 0 && isTopSwipeEnabled()) {
                    dragEdge = DragEdge.Top;
                } else if (distanceY < 0 && isBottomSwipeEnabled()) {
                    dragEdge = DragEdge.Bottom;
                } else return;
            }
            setCurrentDragEdge(dragEdge);
        }

        boolean doNothing = false;
        if (mCurrentDragEdge == DragEdge.Right) {
            boolean suitable = (status == Status.Open && distanceX > mTouchSlop)
                    || (status == Status.Close && distanceX < -mTouchSlop);
            suitable = suitable || (status == Status.Middle);

            if (angle > 30 || !suitable) {
                doNothing = true;
            }
        }

        if (mCurrentDragEdge == DragEdge.Left) {
            boolean suitable = (status == Status.Open && distanceX < -mTouchSlop)
                    || (status == Status.Close && distanceX > mTouchSlop);
            suitable = suitable || status == Status.Middle;

            if (angle > 30 || !suitable) {
                doNothing = true;
            }
        }

        if (mCurrentDragEdge == DragEdge.Top) {
            boolean suitable = (status == Status.Open && distanceY < -mTouchSlop)
                    || (status == Status.Close && distanceY > mTouchSlop);
            suitable = suitable || status == Status.Middle;

            if (angle < 60 || !suitable) {
                doNothing = true;
            }
        }

        if (mCurrentDragEdge == DragEdge.Bottom) {
            boolean suitable = (status == Status.Open && distanceY > mTouchSlop)
                    || (status == Status.Close && distanceY < -mTouchSlop);
            suitable = suitable || status == Status.Middle;

            if (angle < 60 || !suitable) {
                doNothing = true;
            }
        }
        mIsBeingDragged = !doNothing;
    }
  ```
  抛开滑动方式,展示方式之后,这个方法就是通过判断滑动角度来判断是否满足swipeLayout的滑动条件.如果满足就返回 true,通知父类不要拦截事件,反之,就任由父类来拦截处理了.

  最后来看下 *onTouchEvent* 方法.
  ```java
  @Override
   public boolean onTouchEvent(MotionEvent event) {
       if (!isSwipeEnabled()) return super.onTouchEvent(event);

       int action = event.getActionMasked();
       gestureDetector.onTouchEvent(event);

       switch (action) {
           case MotionEvent.ACTION_DOWN:
               mDragHelper.processTouchEvent(event);
               sX = event.getRawX();
               sY = event.getRawY();


           case MotionEvent.ACTION_MOVE: {
               //the drag state and the direction are already judged at onInterceptTouchEvent
               checkCanDrag(event);
               if (mIsBeingDragged) {
                   //向父类申请不要拦截触摸事件,注意这里是viewGroup向它的父类申请不要拦截触摸事件
                   getParent().requestDisallowInterceptTouchEvent(true);
                   mDragHelper.processTouchEvent(event);
               }
               break;
           }
           case MotionEvent.ACTION_UP:
           case MotionEvent.ACTION_CANCEL:
               mIsBeingDragged = false;
               mDragHelper.processTouchEvent(event);
               break;

           default://handle other action, such as ACTION_POINTER_DOWN/UP
               mDragHelper.processTouchEvent(event);
       }

       return super.onTouchEvent(event) || mIsBeingDragged || action == MotionEvent.ACTION_DOWN;
   }
  ```
  这个方法与 *onInterceptTouchEvent* 方法一样,也是要判断滑动的条件是否满足才通知父类是否拦截事件.

  最后的最后,我自己在照着swipeLayout自己实现一个简单的侧滑viewGroup的时候,发现调用 *smoothSlideViewTo* 不起作用,开始以为是像网上说的 *invalidate* 在不同手机上无效的问题,后来发现是没有好好理解ViewDragHelper的 *smoothSlideViewTo* 方法原理.
  看下 *smoothSlideViewTo* 方法,我们可以发现内部是通过 *scroller* 来实现的view的移动的,如果我们不继承实现 *computeScroll* 是无法让view移动的.
  最后在swipeLayout中发现了方法 *computeScroll* .
  ```java
    @Override
    public void computeScroll() {
        super.computeScroll();
        if (mDragHelper.continueSettling(true)) {
            ViewCompat.postInvalidateOnAnimation(this);
        }
    }
  ```

到这里,swipeLayout的主要实现就讲完了,回头看一下核心点就是在ViewDragHelper的使用上面,所以还是感觉有以为的同学可以看一下之前写的的一篇 *ViewDragHelper源码分析* 讲解来对照学习.就到这里吧 .

### 参考资料
[ViewDragHelper详解][a273cef2]

  [a273cef2]: http://www.jcodecraeer.com/a/anzhuokaifa/androidkaifa/2014/0911/1680.html "ViewDragHelper详解"
[github androidSwipeLayout][a5014e0d]

  [a5014e0d]: https://github.com/daimajia/AndroidSwipeLayout "github androidSwipeLayout"
