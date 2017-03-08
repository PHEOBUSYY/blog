layout: "post"
title: "android asyncTask源码分析"
date: "2017-01-19 09:25"
---

## android AsyncTask源码分析(android7.0)
  说道AsyncTask相信大家并不陌生,每当涉及到异步线程的的任务处理的时候,我们第一时间就会想到AsyncTask或者Handler.这里AsyncTask主要是android给我们封装了一层异步任务处理流程,方便使用,所以平时大家在UI中执行异步线程的时候会优先使用AsyncTask,今天就通过源码讲解一下AsyncTask的实现原理和注意事项.

### AsyncTask的用法
` ```java`
`	class MainAsyncTask extends AsyncTask<Object,Object,Object>{`
	 
	       @Override
	       protected void onPreExecute() {
	           super.onPreExecute();
	       }
	 
	       @Override
	       protected void onPostExecute(Object o) {
	           super.onPostExecute(o);
	       }
	 
	       @Override
	       protected Object doInBackground(Object... params) {
	           return null;
	       }
	   }
` ````
`上面这个自定义AsyncTask就是我们平时的用法,可以看到主要用到3个方法,其中 *onPreExecute* 和 *onPostExecute* 分别表示任务开始前后的准备和收尾工作,注意这里这两个方法是在UI主线程中执行的.后面的 *doInBackground* 是任务的执行的代码,耗时的任务执行代码放在这里.注意这个方法是在异步线程中执行的.

当我们要使用AsyncTask的时候,通常是直接创建AsyncTask对象,然后调用execute方法,这样AsyncTask会先调用 *onPreExecute* 方法,然后开始执行 *doInBackground* 方法啊,最后任务完成之后回调 *onPostExecute* 方法.
```java
new MainAsyncTask().execute();
```

### AsyncTask源码
先说结论,AsyncTask的内部实际上维护了一个线程池来调配异步任务(FutureTask)的执行,当异步任务执行完成之后就把结果交给内部的 *Handler* 来回调到UI主线程中.
就这么简单,我们要注意的细节有这么几点:
1. 内部线程池的结构类型
2. 3个常用回调方法的调用时机
3. 任务完成之后为什么不能再次调用这个AsyncTask对象重复执行.

我们先从常用的 *execute* 方法入手:
```java
@MainThread
   public final AsyncTask<Params, Progress, Result> execute(Params... params) {
       return executeOnExecutor(sDefaultExecutor, params);
   }
```
这里execute内部又会调用 *executeOnExecutor* 方法,这里的 *sDefaultExecutor* 后面再做解释,只要明白其就是一个线程池就可以:
```java
@MainThread
   public final AsyncTask<Params, Progress, Result> executeOnExecutor(Executor exec,
           Params... params) {
       if (mStatus != Status.PENDING) {
           switch (mStatus) {
               case RUNNING:
                   throw new IllegalStateException("Cannot execute task:"
                           + " the task is already running.");
               case FINISHED:
                   throw new IllegalStateException("Cannot execute task:"
                           + " the task has already been executed "
                           + "(a task can be executed only once)");
           }
       }

       mStatus = Status.RUNNING;

       onPreExecute();

       mWorker.mParams = params;
       exec.execute(mFuture);

       return this;
   }
```
可以看到,这个代码非常非常的简单,就是判断一下 AsyncTask 对象的状态,如果不是 *PENDING* 状态就抛出异常.如果是 *PENDING* 状态就设置为 *RUNNING* 状态,同时调用 *onPreExecute* 方法,最后交给线程池的 *execute* 方法.是不是很清晰.同时在这里发现了我们的第一个回调方法 *onPreExecute* ,注意这里通过方法上面的注解我们发现是运行在主线程中的.

这里传入线程池的是一个 *FutureTask* 对象 *mFuture* ,里面包含了一个 *mWorker* 对象.下面来看传给线程池的 *mWorker* 对象的实现. *mWorker* 对象是一个 *WorkerRunable* 对象
```java
private static abstract class WorkerRunnable<Params, Result> implements Callable<Result> {
      Params[] mParams;
  }
```
可以看到 *WorkerRunnable* 对象就是一个 *Callable* 对象, *Callable* 和 *Runnable* 的区别就是有没有返回值的区别.这里AsyncTask线程池中使用的 *Callable* .
```java
public AsyncTask() {
       mWorker = new WorkerRunnable<Params, Result>() {
           public Result call() throws Exception {
               mTaskInvoked.set(true);
               Result result = null;
               try {
                   Process.setThreadPriority(Process.THREAD_PRIORITY_BACKGROUND);
                   //noinspection unchecked
                   result = doInBackground(mParams);
                   Binder.flushPendingCommands();
               } catch (Throwable tr) {
                   mCancelled.set(true);
                   throw tr;
               } finally {
                   postResult(result);
               }
               return result;
           }
       };

       mFuture = new FutureTask<Result>(mWorker) {
           @Override
           protected void done() {
               try {
                   postResultIfNotInvoked(get());
               } catch (InterruptedException e) {
                   android.util.Log.w(LOG_TAG, e);
               } catch (ExecutionException e) {
                   throw new RuntimeException("An error occurred while executing doInBackground()",
                           e.getCause());
               } catch (CancellationException e) {
                   postResultIfNotInvoked(null);
               }
           }
       };
   }

```
在AsyncTask的构造函数中,完成了 *mWorker* 对象的初始化.然后把 *mWorker* 对象传给了 *mFuture* 对象. 如果对 *FutureTask* 不是很理解的同学可以去搜索一下相关的资料,非常简单. *FutureTask* 只是把要运行的对象和其返回值做了一个封装,方便线程池的回调使用.

在 *mWorker* 的call方法中,我们发现了第二个回调方法 *doInBackground* ,这个方法用来执行异步任务的,这里是在异步线程中执行的.
当任务执行完成之后,会调用 *postResult(result);*
```java
private Result postResult(Result result) {
       @SuppressWarnings("unchecked")
       Message message = getHandler().obtainMessage(MESSAGE_POST_RESULT,
               new AsyncTaskResult<Result>(this, result));
       message.sendToTarget();
       return result;
   }
```
这里可看到Handler的常规使用.来看Handler的实现:
```java
private static class InternalHandler extends Handler {
       public InternalHandler() {
           super(Looper.getMainLooper());
       }

       @SuppressWarnings({"unchecked", "RawUseOfParameterizedType"})
       @Override
       public void handleMessage(Message msg) {
           AsyncTaskResult<?> result = (AsyncTaskResult<?>) msg.obj;
           switch (msg.what) {
               case MESSAGE_POST_RESULT:
                   // There is only one result
                   result.mTask.finish(result.mData[0]);
                   break;
               case MESSAGE_POST_PROGRESS:
                   result.mTask.onProgressUpdate(result.mData);
                   break;
           }
       }
   }

```
这里Handler里面有两个回调,先说 *MESSAGE\_POST\_RESULT* ,这里回到了asyncTask的 *finish* 方法:
```java
private void finish(Result result) {
        if (isCancelled()) {
            onCancelled(result);
        } else {
            onPostExecute(result);
        }
        mStatus = Status.FINISHED;
    }

```
*finish* 方法,有两个点要说明:首先你在外面调用 *cancel* 停止AsyncTask的时候,AsyncTask内部并没有停止,而是会继续执行,直到最后在finish中才做的的状态判断,只是忽略掉了返回结果而已.so,如果cancel了AsyncTask,那么是不会回调 *onPostExecute* 的,同时任务也不是立刻就停止的.如果想要任务马上停止,只能是在 *doInBackground* 方法中来对AsyncTask的状态做判断,如果状态变化了,里面return,这样可以保证任务立刻停止.
还有一个点要注意的是,前面有个问题:为什么AsyncTask执行完成之后不能继续调用execute方法呢.是因为在 *finish* 方法中,把这个AsyncTask对象的状态设置为 *FINISHED* ,在execute方法中第一步中就判断如果状态不是 *PENDING* 就会抛出异常.关键点在这里,所有我们如果还要二次执行任务的话,只能是重新创建一个新的AsyncTask对象了.

回到刚才Handler中,可以看到还有另一个异步回调情况 *MESSAGE\_POST\_PROGRESS* 这个是用来干什么的呢?通过字面理解我么可以发现是用来更新进度说明的.在代码中,找到了这个方法 *publishProgress* .
```java
/**
    * This method can be invoked from {@link #doInBackground} to
    * publish updates on the UI thread while the background computation is
    * still running. Each call to this method will trigger the execution of
    * {@link #onProgressUpdate} on the UI thread.
    *
    * {@link #onProgressUpdate} will not be called if the task has been
    * canceled.
    *
    * @param values The progress values to update the UI with.
    *
    * @see #onProgressUpdate
    * @see #doInBackground
    */
@WorkerThread
   protected final void publishProgress(Progress... values) {
       if (!isCancelled()) {
           getHandler().obtainMessage(MESSAGE_POST_PROGRESS,
                   new AsyncTaskResult<Progress>(this, values)).sendToTarget();
       }
   }
```
仔细看方法上面的注释,这个方法可以在 *doInBackground* 中调用,来更新进度说明.最后这个方法会在主线程中回调 *onProgressUpdate* .
也就是说如果我们复写 *onProgressUpdate* 方法,我们可以在UI主线程中获取到进度更新提示.

上面提出的3个问题中后两个问题我们已经解释了,3个方法的回调时机非常清晰,同时AsyncTask由于最后finish状态原因是无法再次执行的.
最后说说AsyncTask中线程池的实现,这个点由于在不同的android版本的变化会有很多的差异,这里的源码使用是最新的7.0源码:
```java

    /**
     * An {@link Executor} that executes tasks one at a time in serial
     * order.  This serialization is global to a particular process.
     */
  public static final Executor SERIAL_EXECUTOR = new SerialExecutor();
  private static volatile Executor sDefaultExecutor = SERIAL_EXECUTOR;
```
```java
private static class SerialExecutor implements Executor {
       final ArrayDeque<Runnable> mTasks = new ArrayDeque<Runnable>();
       Runnable mActive;

       public synchronized void execute(final Runnable r) {
           mTasks.offer(new Runnable() {
               public void run() {
                   try {
                       r.run();
                   } finally {
                       scheduleNext();
                   }
               }
           });
           if (mActive == null) {
               scheduleNext();
           }
       }

       protected synchronized void scheduleNext() {
           if ((mActive = mTasks.poll()) != null) {
               THREAD_POOL_EXECUTOR.execute(mActive);
           }
       }
   }
```
这里创建了一种叫做 *SerialExecutor* 的线程池,通过字面理解可以知道是一种串行的线程池,也就是说线程是挨个来交给 *THREAD\_POOL\_EXECUTOR* .这里的的 *SerialExecutor* 用到了 *ArrayDeque* 这种双向队列数据结构.只允许在结尾插入,在头部取出.可以看下其中的 *offer* 和 *poll* 方法.
```java
public boolean offer(E e) {
      return offerLast(e);
  }
```
```java
public E poll() {
       return pollFirst();
   }
```
首先offer新的任务到 *SerialExecutor* 中时,判断 *mActive* 是否存在,如果不存在就去从 *ArrayDeque* 中取一个出来执行,每当其中一个任务执行完成后,最终会调用 *scheduleNext* 方法,这样就保证队列中的任务顺序执行了.

最后 *SerialExecutor* 是把代码交给 *THREAD\_POOL\_EXECUTOR* 线程池了,再来看它的实现方法:
```java
private static final int CORE_POOL_SIZE = Math.max(2, Math.min(CPU_COUNT - 1, 4));
   private static final int MAXIMUM_POOL_SIZE = CPU_COUNT * 2 + 1;
   private static final int KEEP_ALIVE_SECONDS = 30;

   private static final ThreadFactory sThreadFactory = new ThreadFactory() {
       private final AtomicInteger mCount = new AtomicInteger(1);

       public Thread newThread(Runnable r) {
           return new Thread(r, "AsyncTask #" + mCount.getAndIncrement());
       }
   };

   private static final BlockingQueue<Runnable> sPoolWorkQueue =
           new LinkedBlockingQueue<Runnable>(128);

   /**
    * An {@link Executor} that can be used to execute tasks in parallel.
    */
   public static final Executor THREAD_POOL_EXECUTOR;

   static {
       ThreadPoolExecutor threadPoolExecutor = new ThreadPoolExecutor(
               CORE_POOL_SIZE, MAXIMUM_POOL_SIZE, KEEP_ALIVE_SECONDS, TimeUnit.SECONDS,
               sPoolWorkQueue, sThreadFactory);
       threadPoolExecutor.allowCoreThreadTimeOut(true);
       THREAD_POOL_EXECUTOR = threadPoolExecutor;
   }

```
这里的 *THREAD\_POOL\_EXECUTOR* 就是一个线程池的常用实现,这里就不展开了.
这里需要说明的是这个 *THREAD\_POOL\_EXECUTOR* 存在的意义,通过上面代码我们可以看到asyncTask的任务是顺序执行的,也就是说你有再多的任务也是一个一个的执行的.
那这样可能对需求会有影响,所以在 *executeOnExecutor* 方法中可以看到,我们是可以传入其他的线程池的,这样就可以同时执行很多任务了,而 *THREAD\_POOL\_EXECUTOR* 就相当于一个预置的多任务线程池,如果有这种需求,你可以直接把 *THREAD\_POOL\_EXECUTOR* 传给 *executeOnExecutor* 方法,这样就可以让任务同步执行了.

在android2.3之前,AsyncTask使用就是 *THREAD\_POOL\_EXECUTOR* 这种线程池模式,不过由于它最对支持128个任务容量,如果超过容量之后就会报错,同时在创建任务的过程中可能会导致ANR,所以从3.0之后就改成现在这种的串行执行的线程池方式了.这个点是要注意的.

如果有大量的异步任务需要执行的,不推荐使用的AsyncTask,毕竟它本身不适合重要大任务量的处理,这个时候应该自己是实现线程池来完成相关的需求.

### 参考资料
[ Android AsyncTask 源码解析][1]

[1]:	http://blog.csdn.net/lmj623565791/article/details/38614699 "Android AsyncTask 源码解析"