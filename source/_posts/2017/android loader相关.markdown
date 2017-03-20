layout: "post"
title: "android loader相关"
date: "2017-02-15 14:16"
---
### android loader相关

今天第一次学习使用loader,由于不想要使用CursorLoader,所以自己实现了一个asyncTaskLoader,结果发现不起作用,根本没有调用callBack的 *onLoadFinished* 方法.这不科学啊,看这官方示例写的呀.
<!--more-->
后来发现忽略了一个关键点,就是在自定义loader的 *onStartLoading* 方法中,有下面几句话:
```java
@Override protected void onStartLoading() {
        if (mApps != null) {
            // If we currently have a result available, deliver it
            // immediately.
            deliverResult(mApps);
        }

        // Start watching for changes in the app data.
        if (mPackageObserver == null) {
            mPackageObserver = new PackageIntentReceiver(this);
        }

        // Has something interesting in the configuration changed since we
        // last built the app list?
        boolean configChange = mLastConfig.applyNewConfig(getContext().getResources());

        if (takeContentChanged() || mApps == null || configChange) {
            // If the data has changed since the last time it was loaded
            // or is not currently available, start a load.
            forceLoad();
        }
    }

```
逻辑也很简单,就是说如果有数据了,直接通过 *deliverResult* 返回数据,如果没有,强制调用 *forceLoad* .
```java
public void deliverResult(D data) {
      if (mListener != null) {
          mListener.onLoadComplete(this, data);
      }
  }
```
在 *deliverResult* 中调用了callback的 *onLoadFinished* 方法.
```java
public void forceLoad() {
      onForceLoad();
  }
  @Override
    protected void onForceLoad() {
        super.onForceLoad();
        cancelLoad();
        mTask = new LoadTask();
        if (DEBUG) Log.v(TAG, "Preparing load: mTask=" + mTask);
        executePendingTask();
    }
```
forceLoad方法会调用内部AsyncTask发起请求,内部调用 *loadInBackground* 完成耗时任务.

那为什么 *CursorLoader* 不需要复写这一堆东西了,是因为在其内部已经帮我们写好了.
下面是 *CursorLoader* 的 *onStartLoading* 方法:
```java
@Override
   protected void onStartLoading() {
       if (mCursor != null) {
           deliverResult(mCursor);
       }
       if (takeContentChanged() || mCursor == null) {
           forceLoad();
       }
   }
```
对这个方法的处理使我们在第一次使用loader的时候要注意的哦,否者你会发现调用了initloader之后没有反应.
