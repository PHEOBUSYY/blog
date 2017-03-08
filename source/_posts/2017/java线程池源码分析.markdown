layout: "post"
title: "java线程池源码分析"
date: "2017-01-04 08:27"
---
## java线程池源码分析
### Executor接口
```java
public interface Executor {
    void execute(Runnable command);
}
```
线程池的基础接口

### ExecutorService接口方法
```java
public interface ExecutorService extends Executor {


    void shutdown();


    List<Runnable> shutdownNow();


    boolean isShutdown();


    boolean isTerminated();


    boolean awaitTermination(long timeout, TimeUnit unit)
        throws InterruptedException;


    <T> Future<T> submit(Callable<T> task);


    <T> Future<T> submit(Runnable task, T result);


    Future<?> submit(Runnable task);


    <T> List<Future<T>> invokeAll(Collection<? extends Callable<T>> tasks)
        throws InterruptedException;

    <T> List<Future<T>> invokeAll(Collection<? extends Callable<T>> tasks,
                                  long timeout, TimeUnit unit)
        throws InterruptedException;

    <T> T invokeAny(Collection<? extends Callable<T>> tasks)
        throws InterruptedException, ExecutionException;


    <T> T invokeAny(Collection<? extends Callable<T>> tasks,
                    long timeout, TimeUnit unit)
        throws InterruptedException, ExecutionException, TimeoutException;
}

```
线程池相关操作的定义接口

### RunnableFuture 接口
```java
public interface RunnableFuture<V> extends Runnable, Future<V> {
    /**
     * Sets this Future to the result of its computation
     * unless it has been cancelled.
     */
    void run();
}
```
通过适配器把两个不相关的对象组合在一起了.
### Future 和 FutureTask
```java
public interface Future<V> {


    boolean cancel(boolean mayInterruptIfRunning);


    boolean isCancelled();

    boolean isDone();


    V get() throws InterruptedException, ExecutionException;


    V get(long timeout, TimeUnit unit)
        throws InterruptedException, ExecutionException, TimeoutException;
}
```
FutureTask是实现返回值的关键类,里面的核心方法就是 *get* 方法,下面来看一下具体实现:
```java
public V get() throws InterruptedException, ExecutionException {
       int s = state;
       if (s <= COMPLETING)
           s = awaitDone(false, 0L);
       return report(s);
   }

   /**
    * @throws CancellationException {@inheritDoc}
    */
   public V get(long timeout, TimeUnit unit)
       throws InterruptedException, ExecutionException, TimeoutException {
       if (unit == null)
           throw new NullPointerException();
       int s = state;
       if (s <= COMPLETING &&
           (s = awaitDone(true, unit.toNanos(timeout))) <= COMPLETING)
           throw new TimeoutException();
       return report(s);
   }

```
很明显的就是通过state的状态来判断是否需要返回结果,还没有完成就进入 *awaitDone* ,目的是为了在结果没有返回的时候阻塞等待.
```java
private int awaitDone(boolean timed, long nanos)
       throws InterruptedException {
       // The code below is very delicate, to achieve these goals:
       // - call nanoTime exactly once for each call to park
       // - if nanos <= 0, return promptly without allocation or nanoTime
       // - if nanos == Long.MIN_VALUE, don't underflow
       // - if nanos == Long.MAX_VALUE, and nanoTime is non-monotonic
       //   and we suffer a spurious wakeup, we will do no worse than
       //   to park-spin for a while
       long startTime = 0L;    // Special value 0L means not yet parked
       WaitNode q = null;
       boolean queued = false;
       for (;;) {
           int s = state;
           if (s > COMPLETING) {
               if (q != null)
                   q.thread = null;
               return s;
           }
           else if (s == COMPLETING)
               // We may have already promised (via isDone) that we are done
               // so never return empty-handed or throw InterruptedException
               Thread.yield();
           else if (Thread.interrupted()) {
               removeWaiter(q);
               throw new InterruptedException();
           }
           else if (q == null) {
               if (timed && nanos <= 0L)
                   return s;
               q = new WaitNode();
           }
           else if (!queued)
               queued = U.compareAndSwapObject(this, WAITERS,
                                               q.next = waiters, q);
           else if (timed) {
               final long parkNanos;
               if (startTime == 0L) { // first time
                   startTime = System.nanoTime();
                   if (startTime == 0L)
                       startTime = 1L;
                   parkNanos = nanos;
               } else {
                   long elapsed = System.nanoTime() - startTime;
                   if (elapsed >= nanos) {
                       removeWaiter(q);
                       return state;
                   }
                   parkNanos = nanos - elapsed;
               }
               // nanoTime may be slow; recheck before parking
               if (state < COMPLETING)
                   LockSupport.parkNanos(this, parkNanos);
           }
           else
               LockSupport.park(this);
       }
   }
```
具体的细节就不说了,最终就是如果状态是运行中的话就阻塞线程等待,直到那边run方法执行完成之后,修改线程状态:
```java
public void run() {
      if (state != NEW ||
          !U.compareAndSwapObject(this, RUNNER, null, Thread.currentThread()))
          return;
      try {
          Callable<V> c = callable;
          if (c != null && state == NEW) {
              V result;
              boolean ran;
              try {
                  result = c.call();
                  ran = true;
              } catch (Throwable ex) {
                  result = null;
                  ran = false;
                  setException(ex);
              }
              if (ran)
                  set(result);
          }
      } finally {
          // runner must be non-null until state is settled to
          // prevent concurrent calls to run()
          runner = null;
          // state must be re-read after nulling runner to prevent
          // leaked interrupts
          int s = state;
          if (s >= INTERRUPTING)
              handlePossibleCancellationInterrupt(s);
      }
  }
```
执行完成之后进入set(result):
```java
protected void set(V v) {
       if (U.compareAndSwapInt(this, STATE, NEW, COMPLETING)) {
           outcome = v;
           U.putOrderedInt(this, STATE, NORMAL); // final state
           finishCompletion();
       }
   }
```
```java
private void finishCompletion() {
      // assert state > COMPLETING;
      for (WaitNode q; (q = waiters) != null;) {
          if (U.compareAndSwapObject(this, WAITERS, q, null)) {
              for (;;) {
                  Thread t = q.thread;
                  if (t != null) {
                      q.thread = null;
                      LockSupport.unpark(t);
                  }
                  WaitNode next = q.next;
                  if (next == null)
                      break;
                  q.next = null; // unlink to help gc
                  q = next;
              }
              break;
          }
      }

      done();

      callable = null;        // to reduce footprint
  }

```
在这里唤醒线程,通知get方法,已经完成处理了,可以获取返回值了.
### ThreadPoolExecutor
ThreadPoolExecutor中的一个难点是在如何通过一个int值来代表两个作用,一个是线程池状态,一个是线程池运行任务的个数.
```java
   private final AtomicInteger ctl = new AtomicInteger(ctlOf(RUNNING, 0));
   private static final int COUNT_BITS = Integer.SIZE - 3;
   private static final int CAPACITY   = (1 << COUNT_BITS) - 1;

   // runState is stored in the high-order bits
   private static final int RUNNING    = -1 << COUNT_BITS;
   private static final int SHUTDOWN   =  0 << COUNT_BITS;
   private static final int STOP       =  1 << COUNT_BITS;
   private static final int TIDYING    =  2 << COUNT_BITS;
   private static final int TERMINATED =  3 << COUNT_BITS;

   // Packing and unpacking ctl
   private static int runStateOf(int c)     { return c & ~CAPACITY; }
   private static int workerCountOf(int c)  { return c & CAPACITY; }
   private static int ctlOf(int rs, int wc) { return rs | wc; }

```
把一个int值的高位3为作为线程池的状态,后面的29位作为线程池的任务个数.
这里的 *COUNT_BITS* 是29,那么 *CAPACITY* 就是 00011111111111111111111111111111 ,这样 *~CAPACITY* 为 11100000000000000000000000000000,这样通过与操作就可以分别处理两个属性了.非常的巧妙.

然后要明白两个重要的属性 *corePoolSize* 和 *maximumPoolSize* . corePoolSize是用来表示当前允许运行的最小任务个数,也叫核心任务数.maximumPoolSize是指线程池运行运行的最大任务数.

正常情况下,如果没有满足 corePoolSize 的话就创建新的 *worker* 然后执行,如果超过了 corePoolSize 的话一般是进入阻塞队列中等待.如果连阻塞队列都满了的话,尝试在创建新的线程来执行任务,前提是运行的任务数不能超过最大的任务数,也就是 maximumPoolSize .这就是两个size的作用.4中不同的线程池类型的区别主要就在这两个参数上面.
举个例子,比如现在总共有12个任务, corePoolSize 为2,阻塞队列容量为5,最大线程数 maximumPoolSize为10,那首先会优先满足corePoolSize ,1号,2号运行,然后剩下的5个也就是,3,4,5,6,7进入阻塞队列,然后8,9,10号创建新线程来执行填满最大线程数,还剩下的11,12就不予执行了.

上面的这个流程就是线程池的核心流程原理,下面来看具体的代码实现:
ThreadPoolExecutor调用任务有两个方法一个是submit,一个是execute,二者的唯一区别就是submit有返回值,但是内部都是调用了execute方法.
```java
public <T> Future<T> submit(Runnable task, T result) {
       if (task == null) throw new NullPointerException();
       RunnableFuture<T> ftask = newTaskFor(task, result);
       execute(ftask);
       return ftask;
   }


   public <T> Future<T> submit(Callable<T> task) {
       if (task == null) throw new NullPointerException();
       RunnableFuture<T> ftask = newTaskFor(task);
       execute(ftask);
       return ftask;
   }

```
这里传入的就是FutureTask对象啦,FutureTask的细节前面已经说过了,先看 *execute* 方法的实现:
```java
public void execute(Runnable command) {
        if (command == null)
            throw new NullPointerException();
        /*
         * Proceed in 3 steps:
         *
         * 1. If fewer than corePoolSize threads are running, try to
         * start a new thread with the given command as its first
         * task.  The call to addWorker atomically checks runState and
         * workerCount, and so prevents false alarms that would add
         * threads when it shouldn't, by returning false.
         *
         * 如果要执行的任务数量小于corePoolSize,直接通过调用addWorker方法来执行任务.
         *
         * 2. If a task can be successfully queued, then we still need
         * to double-check whether we should have added a thread
         * (because existing ones died since last checking) or that
         * the pool shut down since entry into this method. So we
         * recheck state and if necessary roll back the enqueuing if
         * stopped, or start a new thread if there are none.
         *
         * 如果线程池处于运行状态并且任务成功的进入阻塞队列,二次检查线程池的状态,如果状态变为非运行的时候,尝试回退任务
         * 如果运行中的任务个数为0,运行一个空的任务到addWorker
         *
         * 3. If we cannot queue task, then we try to add a new
         * thread.  If it fails, we know we are shut down or saturated
         * and so reject the task.
         * 如果不能入列,那尝试直接通过addWorker运行任务,如果失败说明线程池已经停止了或者已经满了,拒绝任务
         */
        int c = ctl.get();
        if (workerCountOf(c) < corePoolSize) {
            if (addWorker(command, true))
                return;
            c = ctl.get();
        }
        if (isRunning(c) && workQueue.offer(command)) {
            int recheck = ctl.get();
            if (! isRunning(recheck) && remove(command))
                reject(command);
            else if (workerCountOf(recheck) == 0)
                addWorker(null, false);
        }
        else if (!addWorker(command, false))
            reject(command);
    }

```
这里就对应上面的三种情况,核心任务数没满直接addWorker,如果满了就放入阻塞队列中,如果阻塞队列也满了尝试直接运行,那如何运行任务就进入addWorker方法了.
```java
private boolean addWorker(Runnable firstTask, boolean core) {
      /**
      * 通过一个循环标示来控制一个两层的死循环
      * 首先是判断线程池的状态,如果是停止或以上的状态直接返回false
      * 另外就是前面的 *execute* 方法的传入的 *addWorker(null, false);* 这种情况下说明阻塞队列还有任务没有执行完成,所以要继续把阻塞队列里面的任务执行完才可以,这种情况就会进入底下的第二个循环里面.
      * 第二个循环直接更改运行的线程数,如果成功就跳出循环,如果超出最大限制就返回false
      * 如果发现线程池的状态变化了就重新更新一下
      */
       retry:
       for (;;) {
           int c = ctl.get();
           int rs = runStateOf(c);

           // Check if queue empty only if necessary.
           if (rs >= SHUTDOWN &&
               ! (rs == SHUTDOWN &&
                  firstTask == null &&
                  ! workQueue.isEmpty()))
               return false;

           for (;;) {
               int wc = workerCountOf(c);
               if (wc >= CAPACITY ||
                   wc >= (core ? corePoolSize : maximumPoolSize))
                   return false;
               if (compareAndIncrementWorkerCount(c))
                   break retry;
               c = ctl.get();  // Re-read ctl
               if (runStateOf(c) != rs)
                   continue retry;
               // else CAS failed due to workerCount change; retry inner loop
           }
       }
        /**
        * 如果进入这里说明任务可以执行了
        */
       boolean workerStarted = false;
       boolean workerAdded = false;
       Worker w = null;
       try {
          //包装一个worker对象
           w = new Worker(firstTask);
           //这里的Thread是通过线程池构造函数的ThreadFactory来创建出来的,这里的线程就是worker对象用来执行任务的线程对象,这个对象会陪伴着worker从生到死,不断的执行任务,如果阻塞队列中没有任务的时候,它会阻塞挂起,除非设置了keepAliveTime参数.
           final Thread t = w.thread;
           if (t != null) {
               final ReentrantLock mainLock = this.mainLock;
               mainLock.lock();
               try {
                   // Recheck while holding lock.
                   // Back out on ThreadFactory failure or if
                   // shut down before lock acquired.
                   int rs = runStateOf(ctl.get());
                    //运行态或者停止之后但是阻塞队列还有任务没有完成,设置workerAdded true
                   if (rs < SHUTDOWN ||
                       (rs == SHUTDOWN && firstTask == null)) {
                       if (t.isAlive()) // precheck that t is startable
                           throw new IllegalThreadStateException();
                       workers.add(w);
                       int s = workers.size();
                       //设置当前最大的运行任务数
                       if (s > largestPoolSize)
                           largestPoolSize = s;
                       workerAdded = true;
                   }
               } finally {
                   mainLock.unlock();
               }
               if (workerAdded) {
                 //这里调用worker线程的start方法,最后进入了 *runWorker* 方法
                   t.start();
                   workerStarted = true;
               }
           }
       } finally {
           if (! workerStarted)
               addWorkerFailed(w);
       }
       return workerStarted;
   }
```
当调用worker的中线程的start方法实际上进入了runWorker方法中,先看下work对象的实现:
```java
private final class Worker
      extends AbstractQueuedSynchronizer
      implements Runnable
  {
      /**
       * This class will never be serialized, but we provide a
       * serialVersionUID to suppress a javac warning.
       */
      private static final long serialVersionUID = 6138294804551838833L;

      /** Thread this worker is running in.  Null if factory fails. */
      final Thread thread;
      /** Initial task to run.  Possibly null. */
      Runnable firstTask;
      /** Per-thread task counter */
      volatile long completedTasks;

      /**
       * Creates with given first task and thread from ThreadFactory.
       * @param firstTask the first task (null if none)
       */
      Worker(Runnable firstTask) {
          setState(-1); // inhibit interrupts until runWorker
          this.firstTask = firstTask;
          this.thread = getThreadFactory().newThread(this);
      }

      /** Delegates main run loop to outer runWorker. */
      public void run() {
          runWorker(this);
      }

      // Lock methods
      //
      // The value 0 represents the unlocked state.
      // The value 1 represents the locked state.

      protected boolean isHeldExclusively() {
          return getState() != 0;
      }

      protected boolean tryAcquire(int unused) {
          if (compareAndSetState(0, 1)) {
              setExclusiveOwnerThread(Thread.currentThread());
              return true;
          }
          return false;
      }

      protected boolean tryRelease(int unused) {
          setExclusiveOwnerThread(null);
          setState(0);
          return true;
      }

      public void lock()        { acquire(1); }
      public boolean tryLock()  { return tryAcquire(1); }
      public void unlock()      { release(1); }
      public boolean isLocked() { return isHeldExclusively(); }

      void interruptIfStarted() {
          Thread t;
          if (getState() >= 0 && (t = thread) != null && !t.isInterrupted()) {
              try {
                  t.interrupt();
              } catch (SecurityException ignore) {
              }
          }
      }
  }
```
worker是一个AQS的简单实现,其中要注意的就是它的构造函数,首先通过setState(-1)来把对象锁定,防止中断操作,然后就是thread对象,这里是通过ThreadFactory的newThread来创建出来的新线程,这个线程用来调用传入线程池的任务,相当于任务的载体.这个Thread对象就是worker用来执行任务的环境了,它会和worker一起丛生到死.

线程池是是如何减少了线程的创建的呢?
比如我有100个任务,那你直接调用100个任务的start方法相当于创建了100个线程对不对.而交给线程池却不会傻傻的创建100个线程对象,它会创建你设置的 *corePoolSize* 个worker对象,在worker对象的Thread中来执行任务代码,也就是说每个worker对象不仅仅只是运行一个任务就完了,它会不断的运行任务,从阻塞队列中不断的去取任务,直到没有任务给它处理为止.这个时候它会阻塞,一直等着阻塞队列有数据进来.

注意这里说的worker对象不止执行一个任务是理解线程池的关键.通过这样才能减少线程的创建和消耗,重点要理解worker是怎么处理任务的.下面我们来看runWorker方法的实现:

```java
final void runWorker(Worker w) {
        Thread wt = Thread.currentThread();
        Runnable task = w.firstTask;
        w.firstTask = null;
        w.unlock(); // allow interrupts
        boolean completedAbruptly = true;
        try {
          //这里getTask是从阻塞队列中取没有运行的任务,一直到阻塞队列中没有数据为止
            while (task != null || (task = getTask()) != null) {
                w.lock();
                // If pool is stopping, ensure thread is interrupted;
                // if not, ensure thread is not interrupted.  This
                // requires a recheck in second case to deal with
            // shutdownNow race while clearing interrupt
                if ((runStateAtLeast(ctl.get(), STOP) ||
                     (Thread.interrupted() &&
                      runStateAtLeast(ctl.get(), STOP))) &&
                    !wt.isInterrupted())
                    wt.interrupt();
                try {
                  //这个方法是个钩子函数
                    beforeExecute(wt, task);
                    Throwable thrown = null;
                    try {
                      //运行任务
                        task.run();
                    } catch (RuntimeException x) {
                        thrown = x; throw x;
                    } catch (Error x) {
                        thrown = x; throw x;
                    } catch (Throwable x) {
                        thrown = x; throw new Error(x);
                    } finally {
                        afterExecute(task, thrown);
                    }
                } finally {
                    task = null;
                    w.completedTasks++;
                    w.unlock();
                }
            }
            completedAbruptly = false;
        } finally {
            processWorkerExit(w, completedAbruptly);
        }
    }

```
这里就是调用任务的核心所在了,之所以有这么代码判断是为了处理线程池状态的变化的,比如正在运行任务的过程中,线程池调用了shutDown方法或者shutDownNow方法的时候,要协调这些任务的运行状态.
方法中while的判断,如果worker中的task为空的时候,就对应上面addWorker的时候线程池已经停止了,但是在阻塞队列中还是有任务,就得把阻塞队列中的任务取出来处理.这里就调用的是 *getTask* 方法,从阻塞队列中取任务.
```java
private Runnable getTask() {
        boolean timedOut = false; // Did the last poll() time out?

        for (;;) {
            int c = ctl.get();
            int rs = runStateOf(c);

            // Check if queue empty only if necessary.
            if (rs >= SHUTDOWN && (rs >= STOP || workQueue.isEmpty())) {
                decrementWorkerCount();
                return null;
            }

            int wc = workerCountOf(c);

            // Are workers subject to culling?
            boolean timed = allowCoreThreadTimeOut || wc > corePoolSize;

            if ((wc > maximumPoolSize || (timed && timedOut))
                && (wc > 1 || workQueue.isEmpty())) {
                if (compareAndDecrementWorkerCount(c))
                    return null;
                continue;
            }

            try {
              //从阻塞队列中获取一个任务
                Runnable r = timed ?
                    workQueue.poll(keepAliveTime, TimeUnit.NANOSECONDS) :
                    workQueue.take();
                if (r != null)
                    return r;
                timedOut = true;
            } catch (InterruptedException retry) {
                timedOut = false;
            }
        }
    }
```

这样就保证了只要worker活着它就会不断的从阻塞队列取数据并执行,这种阻塞了等待着执行任务的worker成为IdleWorker,也是在后面会被回收的worker对象.在每次执行完成任务之后来判断一下是不是需要回收闲置的worker对象.关键点就在这里的 *workQueue.poll* 和 *workQueue.take* ,poll支持超时,如果在指定的 *keepAliveTime* 的时间内,阻塞队列中还是没有数据,那么就返回null,如果是 *take* 的话,就会阻塞,等待有新的数据进入阻塞队列中.这个关键点是实现后面要讲的 *newCachedThreadPool* 关键所在.也就是在keepAliveTime的时间内,还是没有新的任务,那么这个worker对象就会被回收.
最后,如果是设置了keepAliveTime的话,超时没有取到数据,就会设置timedOut为true,再次进入上面的 *timed && timedOut* 就会满足条件了,然后返回null,回到上面的runWoker的while循环并跳出,交给下面的 *processWorkerExit* 方法.

```java
private void processWorkerExit(Worker w, boolean completedAbruptly) {
        if (completedAbruptly) // If abrupt, then workerCount wasn't adjusted
            decrementWorkerCount();

        final ReentrantLock mainLock = this.mainLock;
        mainLock.lock();
        try {
            completedTaskCount += w.completedTasks;
            workers.remove(w);
        } finally {
            mainLock.unlock();
        }

        tryTerminate();

        int c = ctl.get();
        if (runStateLessThan(c, STOP)) {
            if (!completedAbruptly) {
                int min = allowCoreThreadTimeOut ? 0 : corePoolSize;
                if (min == 0 && ! workQueue.isEmpty())
                    min = 1;
                if (workerCountOf(c) >= min)
                    return; // replacement not needed
            }
            addWorker(null, false);
        }
    }
```
这里通过重入锁锁定之后修改workers集合的数据,把运行完成的worker移除.如果线程池还没到stop的状态,继续调用addWorker执行阻塞队列中的任务.同时还尝试结束线程池,再来看其中的 *tryTerminate* 方法:
```java
final void tryTerminate() {
        for (;;) {
            int c = ctl.get();
            if (isRunning(c) ||
                runStateAtLeast(c, TIDYING) ||
                (runStateOf(c) == SHUTDOWN && ! workQueue.isEmpty()))
                return;
            if (workerCountOf(c) != 0) { // Eligible to terminate
                interruptIdleWorkers(ONLY_ONE);
                return;
            }

            final ReentrantLock mainLock = this.mainLock;
            mainLock.lock();
            try {
                if (ctl.compareAndSet(c, ctlOf(TIDYING, 0))) {
                    try {
                      //空方法,没有实现
                        terminated();
                    } finally {
                        ctl.set(ctlOf(TERMINATED, 0));
                        termination.signalAll();
                    }
                    return;
                }
            } finally {
                mainLock.unlock();
            }
            // else retry on failed CAS
        }
    }
```
如果现在状态是shutDown,但是阻塞队列中还是有任务就退出,不中止线程池.如果核心线程还是有在运行的时候,调用 *interruptIdleWorkers*
```java
private void interruptIdleWorkers(boolean onlyOne) {
      final ReentrantLock mainLock = this.mainLock;
      mainLock.lock();
      try {
          for (Worker w : workers) {
              Thread t = w.thread;
              if (!t.isInterrupted() && w.tryLock()) {
                  try {
                      t.interrupt();
                  } catch (SecurityException ignore) {
                  } finally {
                      w.unlock();
                  }
              }
              if (onlyOne)
                  break;
          }
      } finally {
          mainLock.unlock();
      }
  }

```
如果是空闲线程,也就是没有任务在运行,所以可以获得锁,就把worker所在的线程中断.

到这里,就会引出两个中断方法的差异,shutDown和showDownNow的区别是:shutDown是中断之后,后续的加入的任务就无法直接运行了,已经在运行和阻塞队列中的还是会继续执行,而shutDownNow是直接中断全部的任务,不管是否在队列中,也不管是否在运行;

```java
public void shutdown() {
        final ReentrantLock mainLock = this.mainLock;
        mainLock.lock();
        try {
            checkShutdownAccess();
            advanceRunState(SHUTDOWN);
            interruptIdleWorkers();
            onShutdown(); // hook for ScheduledThreadPoolExecutor
        } finally {
            mainLock.unlock();
        }
        tryTerminate();
    }
```
可以看到这里调用的是 *interruptIdleWorkers* 方法.
```java
public List<Runnable> shutdownNow() {
       List<Runnable> tasks;
       final ReentrantLock mainLock = this.mainLock;
       mainLock.lock();
       try {
           checkShutdownAccess();
           advanceRunState(STOP);
           interruptWorkers();
           tasks = drainQueue();
       } finally {
           mainLock.unlock();
       }
       tryTerminate();
       return tasks;
   }

```
```java
private void interruptWorkers() {
       final ReentrantLock mainLock = this.mainLock;
       mainLock.lock();
       try {
           for (Worker w : workers)
               w.interruptIfStarted();
       } finally {
           mainLock.unlock();
       }
   }

```
然后回到worker的内部interruptIfStarted:
```java
void interruptIfStarted() {
           Thread t;
           if (getState() >= 0 && (t = thread) != null && !t.isInterrupted()) {
               try {
                   t.interrupt();
               } catch (SecurityException ignore) {
               }
           }
       }
```
可以看到这里调用的 *interruptWorkers* 方法.这就是两个shutdown方法的区别所在.一个是中断空闲任务,一个是中断所有任务.


### newFixedThreadPool,newSingleThreadExecutor,newCachedThreadPool,newScheduledThreadPool 四种类型的区别
这4种常用的线程池的创建都交给Executors来完成的,实际上就是创建了不同的ThreadPoolExecutor.
```java
public static ExecutorService newFixedThreadPool(int nThreads) {
      return new ThreadPoolExecutor(nThreads, nThreads,
                                    0L, TimeUnit.MILLISECONDS,
                                    new LinkedBlockingQueue<Runnable>());
  }

```
可以看到核心任务数和最大任务数是一样的,那就是说线程池支持最多有 *nThreads* 个worker在运行,剩下的任务全部放入阻塞中,等待前面的任务完成后才取出来继续执行.

```java
public static ExecutorService newSingleThreadExecutor() {
       return new FinalizableDelegatedExecutorService
           (new ThreadPoolExecutor(1, 1,
                                   0L, TimeUnit.MILLISECONDS,
                                   new LinkedBlockingQueue<Runnable>()));
   }
```
可以看到核心线程和最大线程数都是1,也就是说同一时间只能有一个任务执行,后续的任务挨个按顺序执行.
```java
public static ExecutorService newCachedThreadPool(ThreadFactory threadFactory) {
      return new ThreadPoolExecutor(0, Integer.MAX_VALUE,
                                    60L, TimeUnit.SECONDS,
                                    new SynchronousQueue<Runnable>(),
                                    threadFactory);
  }

```

可以重复使用线程的线程池,直接把所有任务同时创建worker来运行,不管有多少个,同时由于设置了超时keepAliveTime,到了制定的时候就会回收空闲的worker对象,达到回收资源的目的.注意这里只是回收 *空闲* 的worker,正在运行的worker是不会回收的.
```java
public static ScheduledExecutorService newScheduledThreadPool(
            int corePoolSize, ThreadFactory threadFactory) {
        return new ScheduledThreadPoolExecutor(corePoolSize, threadFactory);
    }

```
支持指定时间的任务的线程池,这里内部通过一个 *DelayedWorkQueue* 来完成定时操作,就不多做解释了.



### Callable和Runnable
Callable是具有返回值的Runnable,这是两个区别.
```java
public interface Callable<V> {
    /**
     * Computes a result, or throws an exception if unable to do so.
     *
     * @return computed result
     * @throws Exception if unable to compute a result
     */
    V call() throws Exception;
}
```
```java
public interface Runnable {

    /**
     * Starts executing the active part of the class' code. This method is
     * called when a thread is started that has been created with a class which
     * implements {@code Runnable}.
     */
    public void run();
}
```

### 总结
线程池的实现难点都在于 ThreadPoolExecutor 中,主要是对其中worker工作方式的理解,还有线程的运行和任务的运行的区别.这是关键,底层还是通过CAS来自旋判断线程的运行状态等属性变化,可以看到CAS是java多线程的核心基础.
刚开始完全不理解worker的实现,后来耐着性子看了好几遍别人的博客总于弄明白个大概,不得不说线程池的实现非常的巧妙.首先是一个int表示两个属性,然后是worker的实现思路来减少线程的创建消耗,对线程的理解更进一步.

### 参考资料
[Java线程池ThreadPoolExecutor源码分析][9e00305a]

  [9e00305a]: http://fangjian0423.github.io/2016/03/22/java-threadpool-analysis/ "Java线程池ThreadPoolExecutor源码分析"

[Java并发包源码学习之线程池（一）ThreadPoolExecutor源码分析][d5fb1523]

  [d5fb1523]: http://zhanjindong.com/2015/03/30/java-concurrent-package-ThreadPoolExecutor "Java并发包源码学习之线程池（一）ThreadPoolExecutor源码分析"
